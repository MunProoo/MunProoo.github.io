---
title: 📆개인프로젝트 Refactoring
date : 2024-01-12 23:00
category : [Study, Projects]
tags : [golang, echo, refactoring, architecture, docker, CI/CD]
---

예전에 **`업무보고 관리 (BSMG)`**라는 토이프로젝트를 진행했었다.  
golang과 친해지기 위해 진행했던 작업물이었는데, 다시 보니 너무 중구난방이라 이참에 공부하는 마음으로 Refactoring을 해보고자 했다.

## **💡 변경점**
### 코드의 모듈화
- 이전 코드는 handler에서 DB에 쿼리 전달 코드까지 한 곳에 몰빵했었다.
> 이전 코드

``` go
// 주간 업무보고 카테고리 setting
func setWeekRptCategoryRequest(writer http.ResponseWriter, request *http.Request) *WebErrorResult {
	var err error

	// 부서 개수 세기
	queryString := "SELECT COUNT(*) FROM bsmgPart"
	var count int
	_ = db.QueryRow(queryString).Scan(&count)

	var result BsmgTreeResult
	result.Result.ResultCode = 1
	if count > 0 {
		result.PartTreeList = make([]*PartTree, count)
	}

	queryString = "SELECT part_idx,part_name FROM bsmgPart WHERE part_idx != 0" // 관리자는 제외
	rows, err := db.Query(queryString)
	if err != nil {
		fmt.Println("쿼리문 오류", err)
		return &WebErrorResult{http.StatusOK, ErrorInvalidParameter}
	}
	defer rows.Close()

	var ii = 0
	for rows.Next() {
		var part_idx sql.NullInt32
		var part_category sql.NullString

		err := rows.Scan(&part_idx, &part_category)
		if err != nil {
			fmt.Println("서버 변수 DB 데이터 할당 오류", err)
			return &WebErrorResult{http.StatusOK, ErrorInvalidParameter}
		}
		result.PartTreeList[ii] = &PartTree{}
		result.PartTreeList[ii].Label = part_category.String
		result.PartTreeList[ii].Value = "1-" + strconv.Itoa(int(part_idx.Int32))
		result.PartTreeList[ii].Parent = "1"
		ii++
	}
	result.PartTreeList[ii] = &PartTree{}
	result.PartTreeList[ii].Label = "부서별 주간 업무보고"
	result.PartTreeList[ii].Value = "1"
	result.PartTreeList[ii].Parent = "0"

	result.Result.ResultCode = 0
	responseMsg, _ := json.Marshal(result)
	sendResultEx(writer, request, http.StatusOK, responseMsg)
	return nil
}
```   
<hr>
기존의 이런 코드들을 모두 목적에 맞게 나누어 모듈에서 불러오도록 변경하였다.   
> 변경한 코드

``` go
// 주간 업무보고 카테고리 정보
func getPartTree(c echo.Context) (err error) {
	log.Println("getPartTree")

	var apiResponse define.BsmgTreeResult

	server, _ := c.Get("Server").(*server.ServerProcessor)
	server.Mutex.Lock()
	defer server.Mutex.Unlock()

	apiResponse.PartTreeList, err = server.DBManager.DBGorm.MakePartTree()
	if err != nil {
		log.Printf("%v \n", err)
		apiResponse.Result.ResultCode = define.ErrorDataBase
		return c.JSON(http.StatusOK, apiResponse)
	}

	apiResponse.Result.ResultCode = define.Success
	return c.JSON(http.StatusOK, apiResponse)
}
```   
> 모듈화에 대해 가슴깊이 이해하지 못했던 것 같다. 분리시키느라 정말 힘들었다.   

<br>

### 프레임워크 사용
- 프레임워크를 무조건적으로 써야하는 건 아니지만, 개발에 필요한 많은 기능을 편하게 제공하는 측면과 코드의 간결성을 위해서 사용하고 싶었다.  

### Mutex를 통한 상호 배제
- 별거아닌 코드로 인해 데드락의 무서움을 실제로 겪어봤다. Mutex는 조심히 다뤄야겠다는 교훈을 얻었다.   

### 암호화 적용
- 기존에는 비밀번호 등의 개인정보를 그냥 저장했었다. 이번에는 SHA256방식보다 보안성이 좋은것으로 알려진 argon2 암호화 방식을 써보았다.   

### JWT 사용
- 세션이 아닌 JWT를 사용해보고싶었다.   
- 토큰을 하나만 만들다가, AccessToken, RefreshToken 2개의 토큰을 만드는 방식으로 변경하였다.   

### Nginx 사용
- 리버스 프록시를 위해 Nginx를 거쳐서 API서버로 가도록 하였다.

### 도커를 통한 컨테이너로 배포
- 서비스를 실제로 도커 이미지화하여 배포까지 해보는 과정을 하고 싶었다.  
- 이 과정에서 무수한 시행착오를 겪게 되었다.. 단순 로컬 개발환경에서 실행, 로컬에서 컨테이너로 실행, 클라우드 서버에 배포까지 고려해야 할 점이 많아 공부를 많이 해야했다.
> [개발자도-CPU-아키텍처를-구분해야합니다.](https://velog.io/@480/%EC%9D%B4%EC%A0%9C%EB%8A%94-%EA%B0%9C%EB%B0%9C%EC%9E%90%EB%8F%84-CPU-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98%EB%A5%BC-%EA%B5%AC%EB%B6%84%ED%95%B4%EC%95%BC-%ED%95%A9%EB%8B%88%EB%8B%A4)


## **😭 아쉽고 미흡한 점**   
### TDD
- 테스트 기반 개발에 대해 알아보기도 했고, 이번에는 그런식으로 해봐야지 했지만, 막상 작성이 쉽지 않아 누락하기 일쑤였다.   
- 또한, TDD는 아무것도 안된 케이스부터 차근차근 단계를 밟아가며 개발해 나가야하는데, 습관적으로 기능 개발 -> 테스트 코드 작성이 되었다. 
- **Github Actions**를 통한 WorkFlow문서를 작성하며, 이부분이 참 아쉬웠다.   
테스트 코드만 잘 짜놨으면, 빌드 오류, 기능 오류를 자동 테스트할 수 있었을 텐데..    

### 스레드의 사용   
- go루틴과 채널링을 통해서 go만의 장점을 강화해보려 했다.
하지만 기존 기능에선 `주간 업무보고 자동 생성` 말고는 할만한 기능이 없었다.  
DB작업이 그나마 고려대상이었는데, DB접근 시 Mutex를 사용하여 상호배제를 하니 병렬처리의 의미가 없다고 판단했다.   
- 추후 나 혹은 지인의 다른 서비스와 연동시킨다면, 사용자 동기화 등에서 사용할 여지가 있을 듯하다.   

### 외부 API 연동   
- 네이버 웍스라던가 여러 오픈 API를 사용해서 기능 추가도 고려했었는데, 마음 속 우선순위에서 많이 밀리다가 결국 안하게 되었다.   

### 캐시 서버
- Redis를 통한 캐시 서버를 두어 불필요한 DB 트래픽 비용을 둬보고싶었지만, 아직 미정이다...   

### 디자인 패턴 적용
- 프로젝트에선 **`Adaptor`**, **`Singleton`** 패턴 정도를 써본 것 같다.  
- 사실 무조건적으로 써야하는 거라기보단, 문제 발생의 여지를 줄이는 측면에서 사용하게 되는 거 같다. 
- 디자인패턴 공부는 계속하여 확장성이 좋고 유지보수가 쉬운 서버를 만들고 싶다.   

### API 문서 제작
- Swagger를 통해서 진행하려 했는데 Scalar 얘기도 많이 보이고 둘 중 하나로 고민중이다.   

### 인가
- 인가는 현재 프론트단에서만 구현되어있다.   
그것도 반쪽짜리인 것이, 타인의 보고서를 마구 훔쳐볼 수 있다.  
(다만, 수정,삭제에 대한 것은 설정된 직급 이상만 가능하도록 했다.)    
- JWT는 서버 미들웨어에서 현재 로그인 상태인지 아닌지만 판별한다. 
- user 도메인, report 도메인에 접근할 때 토큰을 통한 인가를 서버에서도 구현하는 것이 필요하다.   

## **💡 후기**
프로젝트 주제의 매력보다는 공부의 느낌으로 Refactoring을 진행했다.   

사실 멋지게 만들어보고 싶었지만, 부족한 면이 많았다.   

크게는 
- **`소프트웨어 아키텍처`**    
- **`디자인 패턴`**   
- **`CI/CD`**    

작게는 
- **`코드를 가독성 있게 짜는 능력`** 
- **`주석을 잘 다는 능력`** 
- 배운 것을 **`바로 정리하여 기록하는 성실함`** 
   
회사 내에서 당연히 잘 해내야하는 것들을 하며 내 능력을 과신하게 된 것 같다.   
이제와 생각하지만 모르는 것들 투성이고 계속 늘어나는 기술스택들과 빠른 트렌드의 변화가 참 무섭게 느껴진다.  
그렇지만 지금은 **`얕고 넓게보다는 깊고 좁게`** 알아야겠다는 생각이다.

**`천천히 그러나 꾸준히`** 라는 말을 모토로 계속 나아가야겠다.  

### 힘들었던 점!!   
1.레거시를 제거하며 기존 기능을 살리는 것   
2.지식의 실천 (기법, 기술 등등..)    
<br>


다른 토이 프로젝트도 해보고 싶어서 이젠 Refactoring을 메인으로 하진 않고, 생각 날 때 하려 한다.   
더 성장하여 멋진 방향으로 발전시키고 싶다!!

> [레포지토리] <https://github.com/MunProoo/bsmgRefactoring>   


##### TODO 공부한 것 포스팅 
1. 클린 아키텍처
2. Echo 프레임워크 선택 이유
3. JWT 
4. 디자인 패턴 관련
5. Docker 이미지 빌드 및 compose 관련 및 CPU 아키텍처 등 주의점
6. 깃허브 액션 
7. blue/green 무중단 배포   
8. 캐시 서버   