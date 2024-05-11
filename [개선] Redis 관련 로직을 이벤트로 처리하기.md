# 문제 상황
투표 로직과 레디스 관련 로직의 결합도가 높아, 레디스에서 문제가 발생할 경우 투표 기능이 정상적으로 처리되어도 사용자에게 예외 메시지를 반환하는 상황이 발생했습니다. 이는 예외가 전파되는 상황으로, 투표 생성 API 호출 시 정상적으로 투표가 생성되었음에도 예외 메시지로 인해 사용자가 생성 성공 여부를 확인하기 어려웠습니다.

## 현재 코드
#### VoteService.createVote()
```java
public Long createVote(final VoteCreateServiceRequest request) {
    final Long memberId = memberUtils.getCurrentMemberId();

    validateItemIds(request.item1Id(), request.item2Id());

    final Vote vote = voteAppender.append(memberId, request.toImplRequest());

    voteRedisManager.addRanking(vote.getHobby().toString(), getVoteRedis(vote)); // 레디스에 투표를 추가하는 로직

    return vote.getId();
}
```
#### VoteAppender.append()
```java
@Transactional
public Vote append(
    final Long memberId,
    final VoteCreateImplRequest request
) {
    final Vote vote = Vote.builder()
        .memberId(memberId)
        .item1Id(request.item1Id())
        .item2Id(request.item2Id())
        .hobby(request.hobby())
        .content(request.content())
        .maximumParticipants(request.maximumParticipants())
        .build();

    return voteRepository.save(vote);
}
```

<br>

# 요구사항
투표 로직과 레디스 로직을 분리하여, 레디스에 문제가 발생해도 정상적인 응답을 내려줄 수 있도록 변경해야 합니다.

<br>

# 해결 방법
투표 로직과 레디스 관련 로직의 결합도를 낮추기 위해 레디스 로직을 이벤트로 처리하고, 예외가 전파되지 않도록 비동기로 처리하여 별도의 스레드에서 실행되도록 변경하였습니다.


<br>

# 고려 사항
## 🤔 @TransactionalEventListener 사용하지 않은 이유
- @TransactionalEventListener는 트랜잭션이 커밋된 후에 이벤트를 실행하는 기능을 제공하지만, VoteService의 메소드들에 @Transactional 어노테이션이 붙어있지 않기 때문에 작동하지 않습니다.
- VoteAppender.append() 메서드에 @Transactional가 붙어있어서, 투표 로직이 DB에 반영된 후 이벤트가 실행되므로, @TransactionalEventListener를 사용하는 것과 사실상 같은 결과를 얻습니다.

## 🤔 레디스 예외 발생 시 복구 방안
추가적으로, 레디스 예외 발생 시 복구 방안을 고려하였습니다. 예외 발생 시 예외 로그를 저장하고, 스케줄러를 활용하여 재시도를 통해 레디스에 데이터를 다시 삽입하는 방법을 고려할 수 있습니다. 또한, 사용자는 랭킹에 즉시 반영되지 않을 수 있으므로, 사용자에게 알림을 통해 딜레이가 발생할 수 있음을 안내하는 것이 좋습니다.


<br>

# 이벤트 + 비동기 처리 후
투표 생성 서비스에서는 레디스 관련 로직을 독립된 이벤트로 분리하여 처리합니다. 이를 통해 DB에 투표 정보가 정상적으로 저장된 후, 레디스 랭킹 업데이트 로직이 비동기적으로 수행될 수 있도록 합니다. 이벤트 발행이 실패하더라도, 투표 생성 로직에는 영향을 주지 않으며 사용자에게 정상적인 응답을 제공할 수 있습니다.

#### 수정된 VoteService.createVote()
```java
public Long createVote(final VoteCreateServiceRequest request) {
    final Long memberId = memberUtils.getCurrentMemberId();

    validateItemIds(request.item1Id(), request.item2Id());

    final Vote vote = voteAppender.append(memberId, request.toImplRequest());
    // 비동기 이벤트 발행으로 레디스 로직 분리
    eventPublisher.publishEvent(new RankingAddEvent(String.valueOf(vote.getHobby()), getVoteRedis(vote))); // 변경

    return vote.getId();
	}
```
#### RankingAddEvent 클래스
이벤트 클래스는 레디스에 추가할 랭킹 정보를 담고 있으며, 이를 처리할 이벤트 리스너에게 전달합니다.
```java
public record RankingAddEvent(
	String hobby,
	VoteRankingInfo voteRankingInfo
) {
}
```

#### 비동기 이벤트 리스너 RankingEventListener
@Async 어노테이션을 이용해 비동기적으로 레디스 관련 로직을 처리합니다.
```java
@Component
@RequiredArgsConstructor
public class RankingEventListener {

	private final VoteRedisManager voteRedisManager;

	@Async
	@EventListener
	public void addRanking(final RankingAddEvent event) {
		voteRedisManager.addRanking(event.hobby(), event.voteRankingInfo());
	}
    // ...
}
```

<br>

# 마치며
이번 코드 개선을 통해 비동기와 이벤트 처리 방법을 학습하는 있는 기회가 되었고, @TransactionalEventListener 어노테이션에 대해서도 새롭게 알게되었습니다. 이러한 개선으로 예외가 전파되는 문제를 해결하고, 사용자에게 더 나은 경험을 제공할 수 있게 되었습니다.

<br>

### 참고
- https://www.baeldung.com/spring-async
- https://mangkyu.tistory.com/292
- https://www.nextree.io/spring-event/
- https://findstar.pe.kr/2022/09/17/points-to-consider-when-using-the-Spring-Events-feature/
- https://tecoble.techcourse.co.kr/post/2022-11-14-spring-event/