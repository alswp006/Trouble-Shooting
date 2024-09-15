## 문제 발생 코드

- Java Code
    - DTO
    
    ```java
    @Data
    public class AiShortTravelDTO {
        private float LATITUDE;     // 위도
        private float LONGITUDE;    // 경도
        private String MVMN_NM; // "자가용", "대중교통 등"
        private String GENDER; //"남성", "여성"
        private int AGE_GRP; // 20대면 20, 30대면 30 등등
        private int DAY;
        private int TRAVEL_STYL_1;
        private int TRAVEL_STYL_2;
        private int TRAVEL_STYL_3;
        private int TRAVEL_STYL_5;
        private int TRAVEL_STYL_6;
        private int TRAVEL_STYL_7;
        private int TRAVEL_STYL_8;
        private int TRAVEL_MOTIVE_1; // 여행 동기
        private String REL_CD_Categorized; // 동행인 정보
    
        public static AiShortTravelDTO toAiShortTravelDTO(ShortTravelDTO shortTravelDTO){
            AiShortTravelDTO aiShortTravelDTO = new AiShortTravelDTO();
            List<String> travelMotivations = List.of(
                    "일상적인 환경 및 역할에서의 탈출, 지루함 탈피",
                    "쉴 수 있는 기회, 육체 피로 해결 및 정신적인 휴식",
                    "여행 동반자와의 친밀감 및 유대감 증진",
                    "진정한 자아 찾기 또는 자신을 되돌아볼 기회 찾기",
                    "SNS 사진 등록 등 과시",
                    "운동, 건강 증진 및 충전",
                    "새로운 경험 추구",
                    "역사 탐방, 문화적 경험 등 교육적 동기",
                    "특별한 목적(칠순여행, 신혼여행, 수학여행, 인센티브여행)",
                    "기타"
            );
    
            aiShortTravelDTO.LATITUDE = shortTravelDTO.getLocation().getLatitude();
            aiShortTravelDTO.LONGITUDE = shortTravelDTO.getLocation().getLongitude();
            aiShortTravelDTO.DAY = shortTravelDTO.getDay_duration();
            if (shortTravelDTO.getTransport().equals("차량 이용")){
                aiShortTravelDTO.MVMN_NM = "자가용";
            } else{
                aiShortTravelDTO.MVMN_NM = "대중교통 등";
            }
            aiShortTravelDTO.GENDER = shortTravelDTO.getGender();
            aiShortTravelDTO.AGE_GRP = Integer.parseInt(shortTravelDTO.getAgeGroup().replace("대", "").replace(" 이상", "").replace(" 미만", ""));
            aiShortTravelDTO.TRAVEL_STYL_1 = Integer.parseInt(shortTravelDTO.getPreferences().getNature());
            aiShortTravelDTO.TRAVEL_STYL_2 = Integer.parseInt(shortTravelDTO.getPreferences().getDuration());
            aiShortTravelDTO.TRAVEL_STYL_3 = Integer.parseInt(shortTravelDTO.getPreferences().getNewPlaces());
            aiShortTravelDTO.TRAVEL_STYL_5 = Integer.parseInt(shortTravelDTO.getPreferences().getRelaxation());
            aiShortTravelDTO.TRAVEL_STYL_6 = Integer.parseInt(shortTravelDTO.getPreferences().getExploration());
            aiShortTravelDTO.TRAVEL_STYL_7 = Integer.parseInt(shortTravelDTO.getPreferences().getPlanning());
            aiShortTravelDTO.TRAVEL_STYL_8 = Integer.parseInt(shortTravelDTO.getPreferences().getPhotography());
            for (int i = 0; i < travelMotivations.size(); i++){
                if (travelMotivations.get(i).equals(shortTravelDTO.getTravelPurpose())){
                    aiShortTravelDTO.TRAVEL_MOTIVE_1 = i + 1;
                    break;
                }
            }
            aiShortTravelDTO.REL_CD_Categorized = shortTravelDTO.getCompanion().replace("와", "").replace("과", "");
    
            return aiShortTravelDTO;
        }
    }
    
    @Data
    public class RecommendationDTO {
        // "1일차", "2일차"
        private String day;
        // 해당 날짜의 추천 장소 리스트
        private List<PlaceDTO> places;
    }
    ```
    
    - Controller
    
    ```java
    @PostMapping(value = "/submit", consumes = "application/json")
    public List<RecommendationDTO> submitTravelPlan(@RequestBody ShortTravelDTO shortTravelDTO) {
        return aiRecommendationService.getRecommendations(shortTravelDTO);
    }
    ```
    
    - aiRecommendationService.getRecommendations(shortTravelDTO)는 프론트에서 받은 Data를 AI 서버에서 원하는 형태로 가공하고 전달하여 단기 일정 데이터를 받는 메소드
- Python Code
    - app.py
    
    ```python
    from fastapi import FastAPI, HTTPException
    from pydantic import BaseModel
    from typing import List, Dict
    import pickle
    import pandas as pd
    from utils import process_user_input, filter_and_merge, predict_recommendations
    from evaluate import main
    
    app = FastAPI()
    
    # 모델 및 데이터 파일 경로
    model_path = '../models/catboost_model.pkl'
    info_path = '../data/attraction_info.csv'
    
    # 사용자 입력을 위한 Pydantic 모델 정의
    class UserInput(BaseModel):
        LATITUDE: float
        LONGITUDE: float
        MVMN_NM: str  # 이동수단
        GENDER: str  # 성별
        AGE_GRP: int  # 나이
        DAY: int
        TRAVEL_STYL_1: int
        TRAVEL_STYL_2: int
        TRAVEL_STYL_3: int
        TRAVEL_STYL_5: int
        TRAVEL_STYL_6: int
        TRAVEL_STYL_7: int
        TRAVEL_STYL_8: int
        TRAVEL_MOTIVE_1: int  # 여행 동기
        REL_CD_Categorized: str  # 동행자 정보
    
    # 결과를 5개씩 나누어 일차별로 반환하는 함수
    def split_recommendations(recommendations, day_count):
        result = []
        # 5개씩 끊어서 리스트 생성
        for i in range(0, len(recommendations), 5):
            day_recommendations = recommendations[i:i+5]
            day = {"day": f"{(i // 5) + 1}일차", "places": day_recommendations}
            result.append(day)
        return result
    
    # API 엔드포인트 구현
    @app.post("/ai/trip/short", response_model=List[dict])
    def get_recommendations(user_input: UserInput):
        try:
            # 입력 데이터를 처리하는 부분
            user_df = user_input.dict()
    
            # evaluate.py의 main 함수 호출
            recommendations = main(user_input=user_df, model_path=model_path, info_path=info_path)
    
            # 데이터 검증 및 NaN, Infinity 값 처리
            for recommendation in recommendations:
                for key, value in recommendation.items():
                    if isinstance(value, float) and (pd.isna(value) or value in [float('inf'), float('-inf')]):
                        recommendation[key] = 0.0  # NaN이나 Inf를 0으로 대체하거나 원하는 값으로 대체
    
            # 추천 결과를 5개씩 나누어 반환
            split_result = split_recommendations(recommendations, user_input.DAY)
    
            # 리스트 형태로 반환
            return split_result
    
        except Exception as e:
            raise HTTPException(status_code=500, detail=str(e))
    ```
    

## 문제 상황

- AI기반 여행 일정 추천 앱 개발 중 백엔드 단에서는 422에러, Postman에서는 403 에러 발생
- 그리고 403 에러는 Forbidden 에러로 뜻은 아래와 같음
    
    ```java
    웹 페이지를 볼 수 있는 권한이 없습니다.
    ```
    
- 422 에러는 **Unprocessable Entity** 에러로 뜻은 아래와 같음
    
    ```java
    처리할 수 없는 엔터티 응답 상태 코드는 서버가 요청 엔터티의 콘텐츠 유형을 이해하고
    요청 엔터티의 구문이 정확하지만 포함된 명령을 처리할 수 없음을 나타냅니다 
    ```
    
- 보통 403 에러는 cors 문제, 422 에러는 서버 간 주고 받는 데이터의 형식이 다를 때 많이 발생한다고 함

## 해결 1 → 실패

- 우선 403 에러를 해결
- Fast API에서 CORS 설정
- [app.py](http://app.py) 상단에 아래 코드 추가

```java
# 모든 경로 접속 권한 허가 코드
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],  # 모든 형태의 요청 허가
    allow_headers=["*"],
)

```

- 보안 상 모든 출처에 대해 권한을 허가하는 것은 좋지않지만 해당 문제로 인한 에러인지 확인하기 위해 추가
- 달라진게 하나도 없음.

## 해결 2 → 삽질하다가 주고 받는 데이터를 소문자로 변환

- 진짜 거의 10시간동안 혼자 삽질해보다가 주고받는 data의 변수명으로 소문자로 변환하니까 됐음
- Python Code의 받는 변수명 변경
    
    ```python
    class UserInput(BaseModel):
        latitude: float
        longitude: float
        mvmn_nm: str  # 이동수단
        gender: str  # 성별
        age_grp: int  # 나이
        day: int
        travel_styl_1: int
        travel_styl_2: int
        travel_styl_3: int
        travel_styl_5: int
        travel_styl_6: int
        travel_styl_7: int
        travel_styl_8: int
        travel_motive_1: int  # 여행 동기
        rel_cd_categorized: str  # 동행자 정보
    ```
    
- AI 서버에서 사용할 때는 대문자로 사용해야한다고 해서 이후 대문자로 다시 매핑
    
    ```python
    @app.post("/ai/trip/short", response_model=List[dict])
    async def get_recommendations(user_input: UserInput):
        try:
            # 입력 데이터를 처리하는 부분
            user_df = {
                "LATITUDE": user_input.latitude,
                "LONGITUDE": user_input.longitude,
                "MVMN_NM": user_input.mvmn_nm,
                "GENDER": user_input.gender,
                "AGE_GRP": user_input.age_grp,
                "DAY": user_input.day,
                "TRAVEL_STYL_1": user_input.travel_styl_1,
                "TRAVEL_STYL_2": user_input.travel_styl_2,
                "TRAVEL_STYL_3": user_input.travel_styl_3,
                "TRAVEL_STYL_5": user_input.travel_styl_5,
                "TRAVEL_STYL_6": user_input.travel_styl_6,
                "TRAVEL_STYL_7": user_input.travel_styl_7,
                "TRAVEL_STYL_8": user_input.travel_styl_8,
                "TRAVEL_MOTIVE_1": user_input.travel_motive_1,
                "REL_CD_Categorized": user_input.rel_cd_categorized
                }
    		# .. 이후 로직
    ```
    
- Java Code
    
    ```python
    @Data
    public class AiShortTravelDTO {
        private float latitude;     // 위도
        private float longitude;    // 경도
        private String mvmn_nm; // "자가용", "대중교통 등"
        private String gender; //"남성", "여성"
        private int age_grp; // 20대면 20, 30대면 30 등등
        private int day;
        private int travel_styl_1;
        private int travel_styl_2;
        private int travel_styl_3;
        private int travel_styl_5;
        private int travel_styl_6;
        private int travel_styl_7;
        private int travel_styl_8;
        private int travel_motive_1; // 여행 동기
        private String rel_cd_categorized; // 동행인 정보
    }
    ```
    

## 회고

- 정확한 문제는 모르겠지만 Spring 서버의 DTO가 Python 서버로 넘어갈 때 이상하게 변환되는 문제가 생긴 듯
- 이걸로 10시간 넘게 고생하니 현타가 아주 대단…
- 담부터는 이런 문제 안생기게 소문자로 데이터 주고받아야지..
