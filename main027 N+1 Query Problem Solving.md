# MAIN027 N+1 Query Problem Solving

***

## 들어가기 전에

* 저희 팀은 각자 맡은 파트의 세부적인 내용까지 타 팀원에게 공유가 될 수 있도록 메뉴얼 수준의 자세한 설명서를 만들었습니다. 
  * 프로젝트 중 인원에 공백이 생겨 자리를 메워야 하는 상황이 발생했을 때, 프로젝트가 끝나고 타 팀원이 제 코드를 공부할 때 도움을 주기 위한 목적입니다.
* 이 글에서는 main027(에코그린 서울) 팀이 N+1 문제를 해결하여 쿼리최적화를 이루어 낸 과정에 대한 내용이 주를 이룹니다.
* 문제를 해결하면서 든 개인적인 생각 등도 함께 적혀있습니다.



## 목차

1. [N+1 문제 발견][n+1-문제-발견]
1. [해결 과정][해결-과정]
1. [BatchSize 사용][batchsize-사용]
1. [@EntityGraph 사용][@entitygraph-사용]
1. [추가 에러 핸들링][추가-에러-핸들링]
1. [N+1 회고][n+1-회고]

***





## N+1 문제 발견

![문제 발견](/Users/wonyong/codestates/seb41_main_027_blogging/img/문제 발견.png)

 팀원과 함께 장소 검색(searchPlace) 메서드를 만들고 있었다. 우리가 구현하고자 한 장소 검색 메서드는 사용자가 keyword를 입력하면 DB에서 사용자가 입력한 String 값이 포함된 장소명을 모두 불러오는 것이었다. 그러나 단순히 JPA를 이용해서는 검색 조건이 포함된 쿼리를 만들 수 없었다. DB에서 필요한 데이터만을 불러오려면 SQL을 사용하여 직접 검색 조건문을 넣어줘야 했다. 우리는 JPQL을 사용하여 검색 조건문을 추가하고 쿼리가 날아가는 것을 직접 보기 위해 `application.yml` 파일의 show-sql을 true로 바꾸었다. 성공적으로 메서드 구현이 끝나고 show-sql이 true인 상태를 유지한 채로 냅두었다. 이후에 api 요청 비율이 가장 높을 것이라고 예상된 장소 전체 조회 메서드 테스트를 진행하는데 이게 무슨일인가.. 쿼리가 말도안되게 많이 날아가는게 발견되었다. show-sql을 true로 설정하지 않았다면 놓칠 뻔했었다.



 이후 멘토링 시간 때 멘토님께 말씀드렸더니 멘토님께서 N+1 문제에 대해 스터디를 진행해보라는 과제를 내주셨다. N+1 문제에 대해 알아보니 우리 프로젝트에서 가지고 있는 문제와 동일했다. 해결하는 방법도 같이 알아보았는데 여러 방법이 있었고, 바로 팀프로젝트에 적용하는 것보다는 우선 연습삼아 새로운 프로젝트를 만들고 실험해보기로 했다. 스터디를 통해 실습한 내용을 팀원들과 공유하고, 관련 내용을 정리하여 블로깅했다.([링크][https://suzuworld.tistory.com/177]) 시간이 지나 성능 개선에 대한 이야기가 팀회의에서 나왔다. 그때 쿼리 최적화도 함께 진행하자고 의견을 냈고 모두 동의했다.





## 해결 과정

 우선 우리 팀의 연관 관계 매핑은 모두 지연 로딩으로 설정되어 있었다. 매번 참조된 모든 객체를 불러오는 즉시 로딩보다 실제 객체가 필요한 시점에 불러오는 지연 로딩을 사용하는 것이 전체 쿼리의 양을 줄일 수 있기 때문이다. 쿼리의 양을 줄이면 데이터베이스의 성능 향상에도 도움이 되며, 무의미한 네트워크 트래픽 비용을 줄일 수 있다. 따라서 맨 처음에 확인한 것은 지연 로딩이 설정된 시점에서 실제 객체가 호출되는 필드가 무엇이 있는지를 확인하는 것이었다.



### Place 클래스

```java
@Data
@Entity
public class Place extends BaseTime implements Serializable {
  
  ...
    
  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "category_id")
  private Category category;

  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "member_id")
  private Member member;

  // place가 삭제되면 해당 place의 리뷰는 삭제되는게 맞으므로 cascade = REMOVE
  @OneToMany(mappedBy = "place", cascade = CascadeType.REMOVE) // default : LAZY
  private List<Review> reviewList = new ArrayList<>();

  // place가 삭제되면 해당 palce가 속해 있는 북마크는 삭제된다.
  @OneToMany(mappedBy = "place", cascade = CascadeType.REMOVE) // default : LAZY
  private List<Bookmark> bookmarkList = new ArrayList<>();

  @OneToMany(mappedBy = "place", cascade = CascadeType.REMOVE) // default : LAZY
  private List<PlaceLikeUser> placeLikeUserList = new ArrayList<>();
  
  ...
```

* 확인해보니 'category', 'bookmarkList', 'placeLikeUserList' 세 개의 필드가 실제 객체를 호출하여 N+1 쿼리가 나가는 것을 알 수 있었다.(위 사진 참고)

* 이 세 개의 필드는 어차피 객체를 호출하니 즉시 로딩으로 바꾸는 것이 상관이 없어보였다. 
  * 장소 전체 조회 API는 우리 프로젝트의 메인 화면에 들어가면 요청을 보내도록 설계되어 있기 때문에 요청 비율이 압도적으로 많을 수 밖에 없다. 따라서 즉시 로딩을 하는 것이 훨씬 효율적으로 보였다.
* 또한, N+1 문제에 대해 공부했을 때 Fetch Join을 써서 해결하는 방법(지연 로딩으로 해결)은 페이징 쿼리를 가져올 수 없다고 하여 즉시 로딩으로 바꾸어 해결하는 방법을 사용해야만 했다.



```java
@Entity
public class Place extends BaseTime implements Serializable {

  ...
    
  @ManyToOne(fetch = FetchType.EAGER) // 시각적으로 구분하기 위해 명시
  @JoinColumn(name = "category_id")
  private Category category;

  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "member_id")
  private Member member;

  // place가 삭제되면 해당 place의 리뷰는 삭제되는게 맞으므로 cascade = REMOVE
  @OneToMany(mappedBy = "place", cascade = CascadeType.REMOVE) // default : LAZY
  private List<Review> reviewList = new ArrayList<>();

  // place가 삭제되면 해당 palce가 속해 있는 북마크는 삭제된다.
  @OneToMany(mappedBy = "place", cascade = CascadeType.REMOVE, FetchType.EAGER) // EAGER로 변경
  private List<Bookmark> bookmarkList = new ArrayList<>();

  @OneToMany(mappedBy = "place", cascade = CascadeType.REMOVE, FetchType.EAGER) // EAGER로 변경
  private List<PlaceLikeUser> placeLikeUserList = new ArrayList<>();

  ...
```

* 이후 고민은 @EntityGraph와 BatchSize 중 어느 것을 사용하여 해결할 지였다.
* 구글링을 통해 알아본 바로는 BatchSize가 일반적으로 사용되고, @EntityGraph는 연관관계 매핑이 복잡해질수록 사용하기 어렵다고 하여 일단 BatchSize부터 적용해보았다.



### BatchClass 사용

![MultipleBagFetchException](/Users/wonyong/codestates/seb41_main_027_blogging/img/MultipleBagFetchException.png)

* `Caused by: org.hibernate.loader.MultipleBagFetchException` 에러가 발생했다.

* 인터넷에 검색해보니 OneToMany로 매핑되어있는 테이블이 두 개 이상있기 때문이었다. 

* JPA에 대해 깊게 공부하지 않고 사용하여 생긴 문제였다. 그렇다고 프로젝트 마감 기간이 얼마 남지 않은 상황에서 깊게 공부할 수도 없었다. 

* 완벽히 이해할 수는 없었지만 구글링을 통해 정확한 원인을 찾을 수 있었다.

  * Bag이라는 컬렉션은 Set처럼 순서가 없고 List와 같이 중복을 허용하는 자료구조이다.
  * 자바 컬렉션 프레임워크에 이 Bag이라는 녀석이 없기 때문에 List로 대체한다.
  * 그런데, Fetch Join을 복수의 컬렉션에 적용하면 카테시안 곱에 의해 중복 데이터가 발생하게 된다. 이 때문에 `MultipleBagFetchException` 에러를 띄우는 것이다.
  * 여러 해결 방법이 있었지만 우리 팀은 간단하게 List를 Set으로 바꿔 적용하기로 했다. `bookmarkList`와 `placeLikeUserList` 중에 하나를 Set으로 설정하면 카테시안 곱에서 벗어나게 되어(중복 데이터가 Set으로 설정하여 사라지기 때문) 문제가 해결될 것이라고 생각했다.

  

```java
@BatchSize(size = 100)
@OneToMany(mappedBy = "place", cascade = CascadeType.REMOVE, fetch = FetchType.EAGER)
private Set<Bookmark> bookmarkList = new HashSet<>();
```

* 일단 `bookmarkList` 하나에만 적용하기로 했다. 잘 모르는 상태에서는 최소한의 코드만 변경하는 것이 좋다고 생각했다.
  * 그러나, 이 판단은 완전히 미스였다. 복수의 컬렉션에 Fetch Join이 적용되었기 때문에 Set으로 바꾸면 해결되는 것이 아니라 해결된 것처럼 보이는 것이다.



![BatchSize 카테시안 곱](/Users/wonyong/codestates/seb41_main_027_blogging/img/BatchSize 카테시안 곱.png)

* 생각대로 쿼리가 나가지 않았다.(초기 쿼리 이외에 추가적으로 하나의 쿼리가 나가 총 4개의 쿼리가 나가야 한다.)
  * (추가) 프로젝트 당시에는 왜 이렇게 쿼리가 나가는지 이해하지 못 했지만 글을 정리하면서 다시 읽어보니, `처음 요청 쿼리 1` + `category` + `placeLikeUserList` + `bookmarkList`로 총 4개의 쿼리가 나가야 하지만 초기쿼리 1 + 3*3 = 9, 총 10개의 쿼리가 나간 것이었다. 카테시안의 곱과 연관이 있어보인다.
  * placeLikeUserList도 Set으로 바꿔 진행했는데 결과는 같았다(10개의 쿼리가 생성되었다).



```yaml
    jpa:
      properties:
        hibernate:
          default_batch_fetch_size: 100
```

* batchSize를 애플리케이션 전체에 적용해보았다.



![default batch size](/Users/wonyong/codestates/seb41_main_027_blogging/img/default batch size.png)

* 이번애는 `category` 필드는 사라졌지만 나머지는 3번씩 나갔다. 도저히 이해가 가지 않아 일단 멈추었다. 원인을 찾아보고 싶었지만 시간이 촉박했기 때문에 @EntityGraph를 사용해보기로 했다.





### @EntityGraph 사용

```java
@TimeTrace
@EntityGraph(attributePaths = {"category", "bookmarkList", "placeLikeUserList"})
@Query("select p from Place p order by p.placeLikeUserList.size desc")
Page<Place> findAllLikeCount(Pageable pageable);
```

* `@EntityGraph`를 적용했다.
  * BatchSize의 문제를 해결하지 못한 이유도 있었지만, 네트워크 트래픽 비용을 줄이는 것을 최우선의 과제로 삼았기 때문에 초기 쿼리 하나만 나가게 만들고 싶었다.



**Hibernate: select place0_.place_id as place_id1_5_0_, bookmarkli2_.bookmark_id as bookmark1_0_1_, placelikeu3_.place_liker_user_id as place_li1_6_2_, category4_.category_id as category1_1_3_, place0_.created_at as created_2_5_0_, place0_.modified_at as modified3_5_0_, place0_.address as address4_5_0_, place0_.category_id as categor10_5_0_, place0_.description as descript5_5_0_, place0_.kakao_id as kakao_id6_5_0_, place0_.latitude as latitude7_5_0_, place0_.longitude as longitud8_5_0_, place0_.member_id as member_11_5_0_, place0_.name as name9_5_0_, bookmarkli2_.member_id as member_i2_0_1_, bookmarkli2_.place_id as place_id3_0_1_, bookmarkli2_.place_id as place_id3_0_0__, bookmarkli2_.bookmark_id as bookmark1_0_0__, placelikeu3_.member_id as member_i2_6_2_, placelikeu3_.place_id as place_id3_6_2_, placelikeu3_.place_id as place_id3_6_1__, placelikeu3_.place_liker_user_id as place_li1_6_1__, category4_.name as name2_1_3_ from place place0_ left outer join bookmark bookmarkli2_ on place0_.place_id=bookmarkli2_.place_id left outer join place_like_user placelikeu3_ on place0_.place_id=placelikeu3_.place_id left outer join category category4_ on place0_.category_id=category4_.category_id order by (select count(placelikeu1_.place_id) from place_like_user placelikeu1_ where place0_.place_id=placelikeu1_.place_id) desc**

* 쿼리가 하나로 줄어들었다.
* `@BatchSize`보다 확실히 쿼리의 양이 줄었지만 매우 긴 쿼리 하나로 줄어들었기 때문에 가독성이 매우 떨어졌고 둘 중에 무엇이 나은지 판단할 수 없었다.
* 그러나, DB에서 조회하는 속도를 줄이는 것보다 무의미한 네트워크 트래픽 비용을 줄이는 것을 최우선으로 두었기 때문에 팀원들과 회의를 통해 `@EntityGraph`를 사용하기로 결정했다.





### 추가 에러 핸들링



![카테시안곱 문제](/Users/wonyong/codestates/seb41_main_027_blogging/img/카테시안곱 문제.png)

 프론트엔드 분께 다급히 연락이 왔다. 문제는 장소에 좋아요를 누를 때 장소마다 다르지만 좋아요를 누른 횟수 x4, x3으로 반영이 된다는 것이다. 이게 무슨 생뚱맞은 소리야..하고 확인해보니 정말이었다. 두레생협이라는 가게는 원래라면 좋아요가 11이어야 했지만 그 4배인 44가 되어있었다.



```java
@OneToMany(mappedBy = "place", cascade = CascadeType.REMOVE, fetch = FetchType.EAGER)
private Set<Bookmark> bookmarkList = new HashSet<>();

@OneToMany(mappedBy = "place", cascade = CascadeType.REMOVE, fetch = FetchType.EAGER)
private List<PlaceLikeUser> placeLikeUserList = new ArrayList<>();
```

* `PlaceLikeUserList`만 리스트로 놔뒀기 때문에 혹시 카테시안 곱과 관련한 문제인가 하는 생각이 들었다.



```java
private Set<PlaceLikeUser> placeLikeUserList = new HashSet<>();
```

* 정답! Set으로 바꿔주니 바로 해결되었다.





## N+1 회고

 N+1 문제 해결은 비교적 쉽게 해결할 수 있었다. 그러나, 해결 과정에서 JPA에 대한 이해 부족으로 여러 해결 방법 중에 어느 하나도 제대로 사용할 수 없었다. 해결은 했지만 찜찜한 느낌이 남아있어 시간이 될 때 제대로 공부할 생각이다. @EntityGraph를 사용한 것이 최선의 선택이었는지는 확신할 수 없다. 그러나, 날아가는 쿼리 수를 줄여 데이터베이스의 부하와 네트워크 트래픽 비용을 감소시킨다는 우선적인 목표는 달성되었다. 분명히 리팩토링의 여지가 충분히 있어보이기 때문에 나중에 JPA, SQL에 대해 깊게 공부 후 다시 코드를 만져볼 생각이다. N+1 문제를 다루는 것을 통해 JPA도 중요하지만 SQL의 중요성을 절실히 깨달았다. SQLD 자격증이 있다고 가볍게 봤는데 다시 공부해야겠다. 이런 단순한 프로젝트에서도 이정도로 복잡해지는데 실무에서는 더욱 복잡할게 분명하기 때문이다.



















