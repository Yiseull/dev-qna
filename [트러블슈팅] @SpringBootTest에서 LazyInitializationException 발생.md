# 문제 상황
## 코드 소개
Vote와 Voter 엔티티는 1:N 관계를 이루며, 양방향 매핑을 통해 연결되어 있습니다.
#### Vote 엔티티
```java
@Entity
public class Vote {
    // ...
    @OneToMany(mappedBy = "vote", cascade = CascadeType.ALL)
	private final List<Voter> voters = new ArrayList<>();

	public void addVoter(final Voter voter) {
		if (!this.voters.contains(voter)) {
			this.voters.add(voter);
		}
	}
```
### Voter 엔티티
```java
@Entity
public class Voter {
    // ...
    @ManyToOne(fetch = FetchType.LAZY)
	@JoinColumn(name = "vote_id", nullable = false)
	private Vote vote;

	public void participate(final Long itemId) {
		this.itemId = itemId;
		this.vote.addVoter(this);
	}
```

투표 참여 기능은 VoteService와 VoteManager 클래스에서 구현되어 있습니다.

### VoteService 클래스
```java
public void participateVote(
		final Long voteId,
		final Long itemId
	) {
		final Long memberId = memberUtils.getCurrentMemberId();
		final Vote vote = voteReader.read(voteId);

		if (!vote.isVoting()) {
			throw new BusinessException(ErrorCode.VOTE_CANNOT_PARTICIPATE);
		}

		if (!vote.containsItem(itemId)) {
			throw new BusinessException(ErrorCode.VOTE_NOT_CONTAIN_ITEM);
		}

		voteManager.participate(vote, memberId, itemId);
	}
```

### VoteManager 클래스
```java
@Transactional
public void participate(
    final Vote vote,
    final Long memberId,
    final Long itemId
) {
    final Voter voter = new Voter(vote, memberId, itemId);

    voter.participate(itemId);
}
```

## 문제 발생
투표 서비스의 '투표 참여' 기능에 대한 통합 테스트 중 LazyInitializationException 예외가 발생했습니다.

### VoteServiceTest 테스트
```java
@Test
@DisplayName("회원은 투표 아이템 중 하나를 선택하여 투표할 수 있다.")
void participateVoteTest() {
    // given
    final Long itemId = vote.getItem1Id();

    given(memberUtils.getCurrentMemberId())
        .willReturn(1L);

    // when
    voteService.participateVote(voteId, itemId);

    // then
    assertThat(vote.getVoters()).hasSize(1);
}
```
```java
org.hibernate.LazyInitializationException: failed to lazily initialize a collection of role: com.programmers.lime.domains.vote.domain.Vote.voters: could not initialize proxy - no Session

	at org.hibernate.collection.spi.AbstractPersistentCollection.throwLazyInitializationException(AbstractPersistentCollection.java:635)
	at org.hibernate.collection.spi.AbstractPersistentCollection.withTemporarySessionIfNeeded(AbstractPersistentCollection.java:218)
	at org.hibernate.collection.spi.AbstractPersistentCollection.readElementExistence(AbstractPersistentCollection.java:336)
	at org.hibernate.collection.spi.PersistentBag.contains(PersistentBag.java:365)
	at com.programmers.lime.domains.vote.domain.Vote.addVoter(Vote.java:106)
	at com.programmers.lime.domains.vote.domain.Voter.participate(Voter.java:53)
	at com.programmers.lime.domains.vote.implementation.VoteManager.participate(VoteManager.java:29)
```
<br>

# 원인 추론
지연 로딩은 Hibernate 세션이 열려 있고 해당 엔티티가 영속 상태일 때만 가능합니다. 문제 상황에서 Vote 엔티티에 대한 처리가 VoteService에서 시작되어 VoteManager로 이어집니다. 하지만, VoteService에는 @Transactional 어노테이션이 붙어 있지 않아서, VoteService에서 Vote 엔티티를 가져올 때 시작된 트랜잭션이 VoteManager로 이어지지 않습니다. 그 결과, VoteManager에서의 처리는 새로운 트랜잭션으로 시작되며, 이 때 Vote 엔티티는 이미 준영속 상태가 되어 영속성 컨텍스트와의 연결이 끊어진 상태입니다.

따라서, Voter 엔티티를 Vote 엔티티에 추가하려 할 때, Vote 엔티티의 voters 컬렉션을 초기화하기 위해 지연 로딩을 시도합니다. 하지만 이 시점에서 이미 세션이 닫혀 있거나 Vote 엔티티가 준영속 상태이기 때문에, 필요한 영속성 컨텍스트에 접근할 수 없어 LazyInitializationException이 발생하는 것입니다.

<br>

## 🤔 왜 운영 환경에서는 정상적으로 작동하나요?
운영 환경에서는 OSIV(Open Session In View)가 활성화되어 있습니다. 이로 인해, 웹 요청이 시작된 시점부터 종료될 때까지 세션이 유지되므로 지연 로딩이 원활하게 작동합니다. OSIV는 일반적으로 웹 요청에 대해서만 활성화되며, 테스트 환경에서는 활성화되지 않습니다. 따라서, 테스트 환경은 OSIV가 꺼져 있는 환경으로 생각하면 됩니다.

운영 코드에서도 OSIV를 비활성화하고 실행해보니, 테스트 환경과 동일하게 LazyInitializationException이 발생하였습니다.

## 🤔 VoteService는 트랜잭션이 없어도 VoteManager은 트랜잭션이 있는데 지연 로딩 안되나요?
findById 메서드로 Vote 엔티티를 조회한 후 트랜잭션이 종료되면, 해당 엔티티는 영속성 컨텍스트와의 연결이 끊어지면서 준영속 상태가 됩니다. 이 상태에서는 Vote 엔티티가 더 이상 영속성 컨텍스트의 관리를 받지 않게 됩니다. 프록시 객체의 초기화는 영속성 컨텍스트가 관리하는 범위 내에서 이루어지기 때문에, 준영속 상태의 엔티티는 프록시 초기화를 위한 영속성 컨텍스트의 도움을 받을 수 없습니다.

따라서, Vote 엔티티가 준영속 상태일 때, 새로운 트랜잭션을 시작하여 VoteManager 내에서 Vote 엔티티를 다루더라도, 이미 준영속 상태인 Vote 엔티티는 영속성 컨텍스트에 의해 관리되지 않습니다. 이 상황에서 프록시 객체를 초기화하려고 시도하면, 영속성 컨텍스트가 필요한 작업을 수행할 수 없기 때문에 LazyInitializationException이 발생하게 됩니다.

## 🤔 왜 서비스에는 @Transactional을 붙이지 않았나요?
서비스에 @Transactional 어노테이션이 붙지 않은 이유는, 우리 팀이 OSIV를 활성화하여 지연 로딩을 사용하기로 결정하였고, 트랜잭션의 범위를 최소화하기로 결정하였기 때문입니다. 그래서 변경이 발생하는 VoteManager에만 트랜잭션이 적용되었습니다.

<br>

# 해결 방안
## 1. 테스트 코드에 @Transactional 적용 ✅
첫 번째 방법은 테스트 코드에 @Transactional을 붙이는 것입니다. 테스트 코드에 @Transactional 어노테이션을 적용하여, 테스트 진행 동안 세션을 유지하는 것입니다. 

## 2. Eager 로딩으로 변경 ❌
Eager 로딩으로 변경을 시도했으나, 테스트 실패가 발생했습니다. 이는 테스트 환경에서 생성한 Vote가 이미 준영속 상태이고, 변경 감지로 Voter와 관계를 맺은 후 트랜잭션이 종료되어도, 기존의 준영속 상태인 Vote는 업데이트 되지 않아서 발생한 문제로 추정됩니다.

## 3. 서비스 코드에 @Transactional 적용 ❌
서비스 코드에 @Transactional을 적용하는 방법도 실패했습니다. 이 역시 Eager 로딩 변경 시와 같은 원인으로 추정됩니다.

<br>

# 결론
현재로서는 첫 번째 방법인 테스트 코드에 @Transactional을 적용하는 방법을 선택했습니다. 물론, 계속해서 더 좋은 방법을 찾아보고, 적절한 해결책을 찾게 되면 즉시 적용할 것입니다.

테스트는 항상 어려운 부분입니다. 생각했던 대로 작동하지 않을 때 당황스러운 순간도 있지만, 테스트 코드를 통과하는 과정은 매우 재미있습니다. 또한, 테스트를 작성함으로써 더 안정적이고 유지보수가 쉬운 코드를 만들 수 있다는 것이 뿌듯합니다. 앞으로도 계속해서 테스트 코드 작성법을 학습하고 많이 작성하여 연습할 계획입니다.

<br>

### 참고하면 좋은 자료 - 테스트 코드에서 @Transactional 사용
- https://jojoldu.tistory.com/761
- https://tobyepril.tistory.com/8