# [1. API 개발 기본](./1.API-develop-basic)

<details> <summary> 1. 회원 등록 API </summary>

### V1: 엔티티를 RequestBody에 직접 매핑
- 문제점
    - 엔티티에 프레젠테이션 계층을 위한 로직이 추가된다.
    - 엔티티에 API 검증을 위한 로직이 들어간다. (@NotEmpty 등등)
    - 실무에서는 회원 엔티티를 위한 API가 다양하게 만들어지는데, 한 엔티티에 각각의 API를 위한
    모든 요청 요구사항을 담기는 어렵다.
    - 엔티티가 변경되면 API 스펙이 변한다.
- 결론
    - API 요청 스펙에 맞추어 별도의 DTO를 파라미터로 받는다.

### V2: 엔티티 대신에 DTO를 RequestBody에 매핑
- `CreateMemberRequest`를 `Member`엔티티 대신에 RequestBody와 매핑한다.
- 엔티티와 프레젠테이션 계층을 위한 로직을 분리할 수 있다.
- 엔티티와 API 스펙을 명확하게 분리할 수 있다.
- 엔티티가 변해도 API 스펙이 변하지 않는다.

> 참고: 실무에서는 엔티티를 API 스펙에 노출하면 안된다!

</details>


<details> <summary> 2. 회원 수정 API </summary>

</details>


<details> <summary> 3. 회원 조회 API </summary>

</details>