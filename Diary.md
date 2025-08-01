## 📜 프로젝트 개요

- 프로젝트 목표: 교환일기 시스템을 활용한 투두리스트 사이트
- 기간/인원: 2025.04.09 ~ 2025.05.02 / 4명
- 개발 환경 및 언어
    - backend
        Spring Boot - 2.7.14
        Thymeleaf - 3.0.0
        Java - 11
        tomcat - 9.0.79
        Spring security - 5.7.10
        
    - frontend
        HTML5
        CSS3
        JavaScript
        JQuery - 3.7.1
        
    - 데이터베이스
        MySql 8.0.33
        
    - 협업 툴 및 기타 API
        Notion
        GitHub
        OpenAI API
        Kakao API
        Naver Mail API

<br /> <br />

## 🛠️ 프로젝트 주요 기능

1. 로그인 및 회원가입
2. 그룹에서 교환일기 및 목표 달성 여부 공유
3. 투두리스트 및 달성률 계산
4. 일기 분석 후 이모지로 저장

<br /><br />

## 👩‍💻 내가 기여한 부분

1. **목표 등록/수정/삭제 기능**
    - 사용자는 목표 생성 시 `Goal` 엔티티에 `내용`, `시작일`, `마감일`, `공개 범위(OpenScope)`를 함께 입력함.
    - `@DateTimeFormat`을 통해 날짜 형식을 변환하고, 연관된 공개 범위는 외래 키 기반으로 `OpenScope` 테이블과 조인 처리.
    - 목표 수정/삭제 시에는 인증된 사용자(Principal) 기반으로 소유자 검증 후, 관련 엔티티와 연동된 `UserAchiv` 데이터도 함께 처리되도록 설계.
    
2. **목표 달성 여부 기록 (일별 이진 상태 저장)**
    - 사용자가 목표 달성 체크를 수행하면 `GoalStatus` 테이블에 `날짜`와 `is_success` 상태(Boolean)가 저장됨.
    - 체크 미실행 시 false로 자동 저장되며, 이미 존재하는 상태는 업데이트 처리되도록 `Optional` 기반 분기 처리.
    - 날짜 기준 중복 상태 저장을 방지하기 위해 `findTodayGoalStatus(goal, date)`로 사전 조회 후 처리.
    
3. **목표 종료 후 통계 처리 및 업적 저장**
    - 목표 마감일 이후, `UserAchivService.insertOrUpdateUserAchiv(goal)`을 호출하여 달성률 계산 및 저장 수행.
    - 목표 기간 동안 누적된 `GoalStatus` 데이터를 기반으로 성공률(`completionRate`) 산출.
        
        ```jsx
        	@Transactional
        	public void insertOrUpdateUserAchiv(Goal goal) {
        		LocalDate start = goal.getStartDate();
        		LocalDate end = goal.getDueDate();
        
        		long totalDays = ChronoUnit.DAYS.between(start, end) + 1;
        		int countTrue = goalSatusService.countStatusDay(goal, start, end);
        		double userAchivCalc = totalDays > 0 ? countTrue / (double) totalDays : 0;
        
        		UserAchiv existing = userAchivRepository.findByGoalId(goal.getId());
        
        		if (existing != null) {
        			existing.setCompletionRate(userAchivCalc);
        			userAchivRepository.save(existing);
        		} else {
        			UserAchiv newAchiv = new UserAchiv();
        			newAchiv.setGoal(goal);
        			newAchiv.setCompletionRate(userAchivCalc);
        			userAchivRepository.save(newAchiv);
        		}
        	}
        ```
        
    - 이 결과는 `UserAchiv`에 기록되며, 유저별 통계를 저장하는 기반 데이터로 사용됨.
    
4. **교환일기 작성 순서 로직 구현**
    1. **그룹 턴 순서 흐름 제어**
        - `group.getCurrentTurn()`과 그룹의 유저 리스트 크기를 기반으로 **원형 큐(Circular Queue)** 방식으로 턴을 순환 처리.
        - 매일 처음 일기 작성 시점에서 `group.getLastTurnDate()`를 확인하고, 당일이 새 날이면:
            - `group.setLastTurnDate(today)`로 날짜를 갱신하고,
            - `(currentTurn + 1) % users.size()` 계산을 통해 다음 유저에게 턴을 부여.
        - 이 구조는 **정적 인덱스 기반 사용자 순환 시스템**으로, 상태 동기화를 통해 중복 작성 방지와 자동화된 순번 제어를 구현함.
    2. **유저 인증 기반 차례 확인 및 접근 제어**
        - `Principal`에서 현재 로그인된 사용자의 이메일을 가져와 사용자 정보(`User`)를 조회함.
        - 현재 턴에 해당하는 유저(`group.getUsers().get(currentTurn)`)와 로그인 사용자의 ID를 비교하여, **본인의 차례일 때만 일기 작성 페이지 접근 허용.**
        - 차례가 아닌 경우에는 리다이렉트 처리와 함께 `"지금은 차례가 아닙니다."` 메시지 전달.
            
            → 이는 **Access Control(접근 제어)** 구현에 해당하며, 사용자 혼란 방지와 UX 측면에서도 중요함.
            
    3. **시작일과 그룹 인원 수 기반 날짜 리스트 생성**
        - 그룹 인원 수만큼 날짜를 거슬러 올라가 `List<LocalDate>`를 구성.
        - 이는 **일기 작성 기준 기간 설정의 기준 데이터**로 사용됨.
        - 사용자의 목표 달성 상태를 해당 날짜 단위로 조회하는 데 활용됨.
    4. **목표 달성 상태 매핑 및 구조화**
        - 사용자 목표 리스트를 순회하며 각 목표에 대해 `GoalStatus` 데이터를 날짜별로 조회.
        - `{목표 ID: {날짜 문자열: 성공 여부(Boolean)}}` 형태의 **2중 맵(Map)**을 구성하여, 뷰(View)에서 구조화된 데이터로 렌더링 가능하게 함.
        - 이는 **시간-상태 기반 목표 매핑 테이블**로, 목표 분석 및 회고 기능 기반 데이터를 구성하는 핵심 구조임.
    5. **데이터 일관성과 동기화 유지**
        - 날짜가 변경되면 그룹의 `currentTurn` 및 `lastTurnDate`를 즉시 업데이트하여 데이터 불일치 방지.
        - 턴이 그룹 유저 수를 초과하는 경우 `currentTurn = 0`으로 **롤백 처리 후 DB 저장 수행**, 순환 구조 유지.

5. **GPT 기반 이모지 감정 요약 기능**
    
    1. **사용자 입력 수 집 및 프롬프트 생성**
        - 사용자가 작성한 일기 본문(`userMessage`)을 수집함.
        - GPT가 원하는 방식으로 요약할 수 있도록 `"이 일기를 이모지 5개만 사용해서 요약해줘"`라는 문장을 덧붙여 하나의 **프롬프트**를 만듦.

    2. **GPT API 요청 구성**
        - `RestTemplate`을 활용하여 **POST 방식으로 OpenAI 서버에 요청**
        - 요청 본문에는 다음 정보를 포함함:
        - `"model": "gpt-3.5-turbo"`
        - `"messages"` 배열: 사용자가 만든 프롬프트 포함
        - 헤더에는 `"Authorization"`에 `Bearer` 형식의 API Key 삽입
        - API Key는 `.properties`에서 `@Value` 어노테이션으로 주입받아 **보안상 분리**

    3. **응답 파싱 및 예외 처리**
        - 응답은 JSON 형태이며, `ObjectMapper`로 역직렬화
        - 응답 경로: `choices[0].message.content`
        - 여기에 GPT가 보낸 이모지 5개 요약이 담겨 있음
        - 파싱 실패나 API 호출 오류가 발생하면 `"😕 요약 실패"`라는 fallback 값을 리턴하도록 예외 처리 적용

    4.  **사용자 인터페이스로 결과 전달**
        - 최종 이모지 요약 결과는 컨트롤러 또는 서비스 응답 형태로 전달됨
        - View에서는 해당 이모지를 일기 작성 화면 등에 출력하여 사용자가 감정을 한눈에 확인할 수 있도록 구성

<br /><br />

# ⚽ 트러블 슈팅

[Diary 트러블슈팅 (1)](https://www.notion.so/2416ee8f3de9803c9b5bc5964a8ca507?pvs=21)


<br /><br />

# 🎥 시연 동영상

https://youtu.be/Hr1llLmd_zo

 

https://youtu.be/vHFGKVR2zNs

<br /><br />

## 👍 좋았던점

1. **복잡한 턴 구조**를 명확히 이해하고, Set → List 전환과 `currentTurn` 설계로 꼬임 문제 해결
2. Thymeleaf 템플릿을 조건부 렌더링으로 **유지보수성을 높임**
3. 날짜 초기화, 목표 달성률 재계산 등 **흐름 기반 디버깅 경험**

<br /><br />

## ✍️ 아쉬웠던 점

1. 초기에 데이터 흐름과 구조에 대한 설계가 부족하여 **예외 케이스가 후반에 터짐**
2. 기능 위주 구현에 치우쳐 **테스트 코드나 검증 로직이 미흡**했던 부분
