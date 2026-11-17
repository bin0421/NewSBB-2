# NewSBB-2

스프링부트 REST API로 글·댓글의 샘플 데이터, 작성, 삭제 기능을 단계별로 구현하고 테스트하는 저장소.

- 참고 : [강의 설명 1~38강](https://www.slog.gg/p/14172), 강사 저장소 [jhs512/p-14172-2](https://github.com/jhs512/p-14172-2)
- 범위 : 샘플 데이터, 글 작성, 글 삭제, 댓글 작성, 댓글 삭제. 기능마다 테스트 케이스 1개. 그 외 기능(조회, 수정 등)은 구현하지 않는다.
- 단계 번호 : 기능 하나 = 단계 하나 = 커밋 하나. 001 부터 005 까지.
- 진행 기록·결정 사항 : [NewSBB-2.txt](NewSBB-2.txt)

## 개발 환경

| 항목 | 버전 |
|---|---|
| Java | 25 (Temurin) |
| Spring Boot | 4.1.2 |
| Gradle | 9.8.2 (Kotlin DSL) |
| DB | H2 (개발 : 파일, 테스트 : 메모리) |
| 테스트 | JUnit 5, MockMvc |

## 개발 일지

각 단계는 다음 형식으로 적는다.

- **링크** : 단어 "링크"를 누르면 해당 커밋 페이지로 이동해 직전 단계와의 코드 변경(diff)을 볼 수 있다
- **작업** : 추가·수정·삭제한 기능, 객체, 클래스, 메서드
- **설명** : 작업의 구현 방식과 동작 원리 (작업에 적은 내용은 반복하지 않는다)

### 001 샘플 글 3건·댓글 5건 생성

- **[링크](https://github.com/bin0421/NewSBB-2/commit/0fa09b04dcbd2e1dafc8b605ca8305bb57f97752)**
- **작업**
  - 추가 : `BaseEntity`, `Post`·`PostComment` 엔티티, `PostRepository`, `PostService.count()`·`write()`·`findAll()`, `Post.addComment()`, `BaseInitData`, `application-dev.yaml`, `application-test.yaml`
  - 추가 : 의존성 `spring-boot-starter-data-jpa`, `spring-boot-h2console`, `h2`, `spring-boot-starter-data-jpa-test`
  - 추가 : 테스트 `BaseInitDataTest.t1` — 글 3건(제목 1~3), 글별 댓글 수 3·2·0, 댓글 합계 5건 확인
  - 수정 : `BackApplication`
    - 이전 : `@SpringBootApplication`
    - 이후 : `@SpringBootApplication` + `@EnableJpaAuditing`
  - 수정 : `application.yaml`
    - 이전 : 애플리케이션 이름만 설정
    - 이후 : `dev` 프로필, `ddl-auto: update`, SQL 출력, `auto-commit: false`
- **설명**
  - `BaseEntity` 는 `@MappedSuperclass` 로 `id`·`createDate`·`modifyDate` 를 모든 엔티티에 물려준다. 날짜는 `AuditingEntityListener` 가 채우고, 이 리스너는 `@EnableJpaAuditing` 이 있어야 동작한다. `equals`·`hashCode` 는 `id` 로 비교한다.
  - `PostComment` 는 `@ManyToOne(fetch = LAZY)` 로 글을 가리키고, `Post.comments` 는 `mappedBy = "post"` 인 양방향 관계다. 댓글은 `Post.addComment()` 로 글의 리스트에 넣기만 하고 저장은 `cascade = PERSIST` 에 맡긴다.
  - `BaseInitData` 는 `ApplicationRunner` 빈으로 앱 구동 직후 실행된다. `@Lazy` 로 주입한 자기 자신(`self`)을 거쳐 `work1()` 을 호출해야 프록시를 통과해 `@Transactional` 이 적용되고, 트랜잭션이 끝날 때 댓글이 함께 INSERT 된다. 글이 이미 있으면 건너뛰어 재시작해도 중복 생성되지 않는다.
  - 테스트는 `test` 프로필의 메모리 DB(`jdbc:h2:mem:db_test`)를 쓴다. 테스트 컨텍스트가 뜰 때도 `BaseInitData` 가 실행되므로 샘플 데이터가 들어간 상태에서 검사한다. `@Transactional` 이 있어 지연 로딩되는 `comments` 를 테스트 안에서 읽을 수 있다.

### 002 글 작성

- **[링크](https://github.com/bin0421/NewSBB-2/commit/fcf4ad8d7d014f6f246bd9a3e71bf69d20e8932f)**
- **작업**
  - 추가 : `ApiV1PostController.write()` — `POST /api/v1/posts`, 요청 본문 `PostWriteReqBody(title, content)`, 응답 `RsData<PostWriteResBody(totalCount, post)>`
  - 추가 : `PostDto`, `RsData`, `ResponseAspect`, `PostService.findLatest()`, `PostRepository.findFirstByOrderByIdDesc()`
  - 추가 : 의존성 `spring-boot-starter-validation`, `spring-boot-starter-validation-test`
  - 추가 : 테스트 `ApiV1PostControllerTest.t1` — 처리한 컨트롤러·메서드, 201, `resultCode`, `msg`, `totalCount`, 작성된 글의 `id`·`title`·`content` 확인
- **설명**
  - `@RequestBody` 가 JSON 본문을 레코드로 바꾸고, `@Valid` 가 `@NotBlank`·`@Size` 제약을 검사한다. 엔티티 대신 `PostDto` 를 응답에 담아 엔티티 구조가 API 에 그대로 드러나지 않게 한다.
  - `RsData` 는 `resultCode`·`msg`·`data` 로 응답 형식을 통일한다. `resultCode` 의 `-` 앞 숫자(`201`)를 `statusCode` 로 계산하고, `statusCode` 는 `@JsonIgnore` 로 JSON 에서 뺀다.
  - `ResponseAspect` 는 컨트롤러의 매핑 메서드가 `RsData` 를 돌려줄 때 끼어들어(`@Around`) `statusCode` 를 HTTP 응답코드로 설정한다. 그래서 액션 메서드는 `ResponseEntity` 없이 `"201-1"` 만으로 201 을 응답한다.
  - 테스트는 `@AutoConfigureMockMvc` 의 `MockMvc` 로 서버 없이 요청을 보내고, 요청(`perform`)과 검증(`andExpect`)을 나눠 적는다. 기대값은 DB 에서 가장 최신 글(`findLatest`)과 전체 글 수를 가져와 응답과 대조한다. `@Transactional` 로 테스트가 끝나면 작성한 글이 롤백된다.

### 003 글 삭제

- **[링크](https://github.com/bin0421/NewSBB-2/commit/a07fcbc6c0252bedc25236c9e471bb796cb089b9)**
- **작업**
  - 추가 : `ApiV1PostController.delete()` — `DELETE /api/v1/posts/{id}`, 응답 `RsData<Void>`
  - 추가 : `PostService.findById()`·`delete()`
  - 추가 : 테스트 `ApiV1PostControllerTest.t2` — 1번 글 삭제 후 처리한 컨트롤러·메서드, 200, `resultCode`, `msg`, 남은 글 수 2, 1번 글 없음 확인
  - 수정 : `Post.comments` 의 cascade
    - 이전 : `cascade = {PERSIST}`
    - 이후 : `cascade = {PERSIST, REMOVE}`
- **설명**
  - 삭제는 조회와 같은 주소에 HTTP 메서드만 `DELETE` 로 바꿔 표현한다. 돌려줄 데이터가 없으므로 `RsData<Void>` 로 `data` 가 비어 있음을 드러낸다. `"200-1"` 이므로 `ResponseAspect` 가 200 으로 응답한다.
  - 댓글 테이블이 글 id 를 외래키로 가지므로 댓글이 달린 글만 지우면 제약 조건 위반이 난다. `CascadeType.REMOVE` 로 글을 지울 때 연결된 댓글을 먼저 DELETE 한다.
  - 테스트 트랜잭션 안에서는 DELETE 가 바로 실행되지 않는다. `count()` 같은 조회 쿼리가 실행되기 전에 JPA 가 자동으로 flush 하므로, 댓글·글 DELETE 가 실제로 DB 에 나가 제약 조건까지 검사된 뒤 롤백된다.

### 004 댓글 작성

- **[링크](https://github.com/bin0421/NewSBB-2/commit/d8ab1a207d646a44c58cf58f95f9bea5c5c4017f)**
- **작업**
  - 추가 : `ApiV1PostCommentController.write()` — `POST /api/v1/posts/{postId}/comments`, 요청 본문 `PostCommentWriteReqBody(content)`, 응답 `RsData<PostCommentDto>`
  - 추가 : `PostCommentDto`, `PostService.writeComment()`·`flush()`
  - 추가 : 테스트 `ApiV1PostCommentControllerTest.t1` — 1번 글에 댓글 작성 후 처리한 컨트롤러·메서드, 201, `resultCode`, `msg`, 작성된 댓글의 `id`·`content`, id 가 0 이 아님 확인
- **설명**
  - 댓글은 글에 딸린 자원이므로 주소를 `/api/v1/posts/{postId}/comments` 로 두고, 클래스 레벨 `@RequestMapping` 의 `{postId}` 를 `@PathVariable` 로 받는다. 저장은 1단계와 같이 `Post.addComment()` 로 글의 댓글 리스트에 넣고 `cascade = PERSIST` 에 맡긴다.
  - 리스트에 넣기만 하면 INSERT 는 트랜잭션이 끝날 때 나가므로, 액션 메서드 안에서는 새 댓글의 id 가 아직 0 이다. `flush()` 로 INSERT 를 미리 실행해 `IDENTITY` 로 생성된 id 를 받은 뒤 응답을 만든다. flush 는 커밋이 아니어서 이후 오류가 나면 롤백된다.
  - 테스트에서 1번 글의 마지막 댓글을 꺼내 응답의 id·메시지와 대조하고, id 가 양수인지 확인해 flush 가 빠지면 실패하도록 했다.

### 005 댓글 삭제

- **[링크](COMMIT_005)**
- **작업**
  - 추가 : `ApiV1PostCommentController.delete()` — `DELETE /api/v1/posts/{postId}/comments/{id}`, 응답 `RsData<Void>`
  - 추가 : `Post.findCommentById()`·`deleteComment()`, `PostService.deleteComment()`
  - 추가 : 테스트 `ApiV1PostCommentControllerTest.t2` — 1번 글의 1번 댓글 삭제 후 처리한 컨트롤러·메서드, 200, `resultCode`, `msg`, DB 에서 다시 읽은 1번 글에 1번 댓글이 없고 댓글이 2건 남았음 확인
  - 수정 : `Post.comments` 의 고아 객체 제거
    - 이전 : `cascade = {PERSIST, REMOVE}`
    - 이후 : `cascade = {PERSIST, REMOVE}, orphanRemoval = true`
- **설명**
  - 댓글은 글을 통해서 찾고 지운다. `findCommentById()` 는 글의 댓글 리스트에서 id 로 찾으므로, 다른 글의 댓글 번호를 주소에 넣으면 찾지 못한다.
  - `deleteComment()` 는 리스트에서 댓글을 빼기만 한다. `orphanRemoval = true` 이면 부모 리스트에서 빠진 자식을 고아로 보고 트랜잭션이 끝날 때 DELETE 한다. 이 변경 감지는 트랜잭션 안에서만 동작하므로 액션 메서드에 `@Transactional` 이 필요하다.
  - 테스트는 `EntityManager` 의 `flush()` 로 DELETE 를 실행하고 `clear()` 로 영속성 컨텍스트를 비운 뒤 글을 다시 조회한다. 메모리 속 리스트가 아니라 DB 에서 정말 지워졌는지를 확인한다.
