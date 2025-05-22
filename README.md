# 시스템 별 현황 데이터 조회
## 1. API 호출 방식
### 장점
 - 외부 DB를 직접 연결하지 않아도 됨 -> 데이터 소스 보호(DB관련 정보)
 - 최신화된 데이터를 조회할 수 있음 -> 하지만 조회할 때마다 최신화된 데이터가 필요한 지 검토해 볼 필요가 있음
 - 외부 DB 접근이 차단된 환경에서도 API를 통해 데이터를 안전하게 가져올 수 있음.
 - 코드 관리 용이

### 단점
 - 대시보드에 접근할 때마다 데이터 조회 시간이 오래 걸릴 수 있음.
 - API 변경 시 영향을 받음
 - 데이터를 실시간으로 업데이트하려면 추가적인 WebSocket 구현이 필요 -> 실시간 데이터가 필요한지 검토해 볼 필요가 있음
 - API 변경 시 시스템 담당자들의 비용이 들어감
---
## 2. 서버에서 직접 DB와 연결하여 데이터 추출하는 방식
### 장점
 - DB와 직접 연결하여 복잡한 변환 로직 없이 데이터를 추출 가능
 - API 호출보다 대량 데이터 처리에서 효율적 -> 시간적 비용이 덜 듦
 - 조회해야 하는 데이터가 추가되거나 변경될 시 시스템 담당자들의 비용이 API 호출 방식보다 덜 들어감
    - API 호출 방식 : ex) 10명의 담당자 -> API 수정 -> 배포 -> 다운타임 발생
    - 직접 DB와 연결 : ex) 10명의 담당자 -> 쿼리 수정 -> 쿼리 전달
    - 수정된 쿼리를 반영하고 발생하는 다운타임은 "현황 시스템"에만 발생

### 단점
 - 외부 DB 스키마 변경 시 로직 수정 필요
 - 외부 DB의 접근 권한 설정이 올바르지 않으면, 데이터 노출 가능성 있음
 - 방화벽 설정, 네트워크 구성 등이 필요
---
## 데이터 추출 방식 선정 전 고려사항
### 1. 실시간 데이터를 보여줘야 할 것인지
   - 경우에 따라 WebSocket 구현이 필요할 수 있음
   - 실시간 데이터가 필요하지 않다면 Spring Batch를 통해 데이터를 적재하고, 대시보드에 간단히 수치로 보여줄 수 있음
     - 각 시스템 메인 운영 시간을 방해하지 않을 수 있음.
     - 현황 데이터 조회가 시스템에 부하를 줄 수 있기 때문에 새벽에 배치가 실행되도록 Spring Batch를 적극 활용할 수 있음
     - Spring Batch를 사용한다면 현황 데이터를 저장할 테이블 생성이 필요함
---
## 방식 별 구조
### 1. API 호출 방식
```plaintext
GetDataInterface        # 데이터 호출 interface
│   ├── system_1_impl   # 시스템1 데이터 호출 구현체
│   ├── system_2_impl   # 시스템2 데이터 호출 구현체
│   └── system_3_impl   # 시스템3 데이터 호출 구현체
ResponseDTO             # 공통으로 사용될 DTO -> 시스템 별로 필드명을 동일하게 할 수 있는지 확인 필요
│
DashboardController     # dashboardController
```
```java
@GetMapping("/dashboard")
    public ModelAndView dashboard2(ModelAndView mv) {
        GetDataInterface impl_1 = new Impl_1();
        ResponseDTO system_1 = impl_1.getData();

        GetDataInterface impl_2 = new Impl_2();
        ResponseDTO system_2 = impl_2.getData();

        mv.addObject("system_1", system_1);
        mv.addObject("system_2", system_2);

    /**
     * 대시보드 내에서 시스템별로 Tab으로 나눠서 하나씩 조회하도록 하는
     * 방법도 있음
     */
        mv.setViewName("dashboard");
        return mv;
    }
```

### 2. 서버에서 직접 DB와 연결하여 데이터 추출하는 방식
```yml
external:
  system_1:
    url: ${SYSTEM_1_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  system_2:
    url: ${SYSTEM_2_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```
```java
public class DbConfig {
    private String url;
    private String username;
    private String password;

    // 생성자 및 Getter/Setter
    public DbConfig(String url, String username, String password) {
        this.url = url;
        this.username = username;
        this.password = password;
    }

    public String getUrl() {
        return url;
    }

    public String getUsername() {
        return username;
    }

    public String getPassword() {
        return password;
    }
}
```
```java
public class ExternalDbManager {
    public static DbConfig getExternalDbConfig(String systemCode) {
        switch (systemCode) {
            //url, id, password는 yml에서 가져옴 -> 각 시스템 url, id, password는 환경변수로 관리
            case "A01":
                return new DbConfig("jdbc:mysql://db1.example.com:3306/system1", "user1", "pass1");
                break;
            case "A02":
                return new DbConfig("jdbc:mysql://db1.example.com:3306/system2", "user1", "pass1");
                break;
        }

        return null;
    }
}
```
```java
public class ExternalDbFetcher {

    public List<UserDto> fetchFromDb(String systemCode) {
        // DB 연결 정보
        DbConfig config = ExternalDbManager.getExternalDbConfig(systemCode);
        if(config == null) throw new Exception();
        
        List<UserDto> userList = new ArrayList<>(); // 결과를 저장할 리스트

        String query = "SELECT id, name, status FROM users WHERE status = ?";

        try (Connection connection = DriverManager.getConnection(config.getUrl(), config.getUsername(), config.getPassword());
             PreparedStatement preparedStatement = connection.prepareStatement(query)) {

            preparedStatement.setString(1, "ACTIVE"); // 파라미터 바인딩

            try (ResultSet resultSet = preparedStatement.executeQuery()) {
                // ResultSet의 각 행을 UserDto로 매핑하여 리스트에 추가
                while (resultSet.next()) {
                    UserDto user = new UserDto(
                            resultSet.getInt("id"),
                            resultSet.getString("name"),
                            resultSet.getString("status")
                    );
                    userList.add(user);
                }
            }
        } catch (Exception e) {
            System.err.println("Error fetching data from DB: " + config.getUrl());
            e.printStackTrace();
        }

        return userList;
    }
}
```
```java
@GetMapping("/dashboard")
    public ModelAndView dashboard(@RequestParam String systemCode, ModelAndView mv) {
        List<UserDto> users = ExternalDbFetcher.fetchFromDb(systemCode);
        
        mv.addObject("users", users);

        mv.setViewName("dashboard");
        return mv;
    }
```
