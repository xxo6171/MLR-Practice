# [고급보건통계학] Multiple Regression 예제
### 이경민, 보건행정학과 통합과정 3학기
<br>


## 주제: 일 하는 사람이 일 하지 않는 사람보다 행복한가?
<br>


## H1: 일 하는 사람이 일 하지 않는 사람보다 행복할 것이다.
<br>


## 자료원 및 대상:

- 노동패널(KLIPS) 26차 자료 (2023년)
    - 단년도
- 대상: 45세 이상



## 종속변수

- 현재 행복도 (0-9점)

## 독립변수

- **[관심변수]** 경제활동상태 ( 미취업자, 취업자 )
- 성별 ( 남, 여 )
- 연령 ( 연속형 )
- 학력 ( 초졸이하, 중졸, 고졸, 대졸이상)
- 거주지역 ( 서울, 광역시, 그 외 또는 지방 )
- 사회경제적지위 ( 하, 중, 상 )
- 현재건강상태 ( 나쁨, 보통, 좋음 )
- 활동제한여부 ( 있음, 없음 )
- 흡연여부 ( 핌, 안 핌 )
- 음주여부 ( 마심, 안 마심 )



## 모델

- Multiple Linear Regression



## 분석 도구

- SAS 9.4



## SAS 구현:

## 0. 프로세스

![image](https://github.com/user-attachments/assets/e5ff173d-f428-414c-9ed2-6842b2a7c1cc)

## 1. 라이브러리 불러오기

```swift
LIBNAME RAW "mypath\RAW";
LIBNAME TEMP "mypath\TEMP";
```

## 2. 데이터 불러오기

```swift
************************************ Load Dataset **************************************************
KLIPS 24th Data (2023)

** PID: person id

** Dependent Variable (continuous)
 * current happiness level ( PH )

** Indepent Variables
 * Work status
  * Interested Group: worker
  * Comparison Group: non-worker
 * Demographic
  * Gender
  * Age
 * Socioeconomic
  * Education
  * Region
  * Socioeconomic Status
 * Health related
  * Current Health Status
  * Activity Limitation - 1: Difficulty in learning, remembering, or concentrating  
  * Activity Limitation - 2: Difficulty in dressing, bathing, or moving around the house  
  * Activity Limitation - 3: Difficulty in going out for shopping or hospital visits  
  * Activity Limitation - 4: Difficulty in job-related activities  
  * Smoking Status  
  * Drinking Status
****************************************************************************************************;

PROC SQL;
CREATE TABLE TEMP.DATA AS
SELECT PID AS PID,
       P260201 AS WORK, PA268141 AS HAPPINESS,
	   P260101 AS GENDER, P260104 AS BIRTH,
	   P260110 AS EDU, P260121 AS REGION, P266615 AS SCE_STATUS,
	   P266101 AS HTH_STATUS,
	   P266106-1 AS LMT_ATV_6, P266107-1 AS LMT_ATV_7, P266108-1 AS LMT_ATV_8, P266109-1 AS LMT_ATV_9,
	   P266158 AS SMK, P266161 AS DRK 
FROM RAW.KLIPS26P;
QUIT;

/*결측 값 확인*/
PROC MEANS DATA=TEMP.DATA N NMISS;
RUN;
```

## 3. 데이터 전처리

```swift
/*Grouping*/
PROC SQL;
CREATE TABLE TEMP.DATA2 AS
SELECT PID, HAPPINESS,

     /* 0: 미취업자; 1: 취업자 */
     CASE WHEN WORK = 2 THEN 0 ELSE 1 
	   END AS WORK,
	   
	   /* 0: 여자; 1: 남자 */
	   CASE WHEN GENDER = 2 THEN 0 ELSE 1 
	   END AS GENDER,

	   /*연령*/
	   2023 - BIRTH AS AGE, 
	   
	   /* 1: 초졸이하; 2:중졸; 3:고졸; 4:대졸이상 */
       CASE WHEN EDU BETWEEN 1 AND 3 THEN 1 
	   		WHEN EDU = 4 THEN 2
			WHEN EDU = 5 THEN 3
		    ELSE 4
	   END AS EDU,
	   
	   /* 1: 서울; 2:광역시; 3: 그 외 또는 지방 */
       CASE WHEN REGION = 1 THEN 1 			
            WHEN REGION BETWEEN 2 AND 7 THEN 2
            ELSE 3
	   END AS REGION,
	   
	   /* 1: 하; 2:중; 3: 상 */
	   CASE WHEN SCE_STATUS BETWEEN 5 AND 6 THEN 1
	        WHEN SCE_STATUS BETWEEN 3 AND 4 THEN 2
			WHEN SCE_STATUS BETWEEN 1 AND 2 THEN 3
	   END AS SCE_STATUS,
	   
	   /* 1: 나쁨; 2: 보통; 3: 좋음 */
	   CASE WHEN HTH_STATUS BETWEEN 4 AND 5 THEN 1
	        WHEN HTH_STATUS = 3 THEN 2
			WHEN HTH_STATUS BETWEEN 1 AND 2 THEN 3
	   END AS HTH_STATUS,
	   
	   /* 0: 일상활동 제한 있음; 1: 일상활동 제한 없음*/
	   CASE WHEN LMT_ATV_6 + LMT_ATV_7 + LMT_ATV_8 + LMT_ATV_9 >= 1 THEN 0 ELSE 1
	   END AS LMT_ATV,
	   
	   /* 1: 담배 핌; 2: 담배 안 핌 */
	   CASE WHEN SMK = 1 THEN 1 ELSE 2
	   END AS SMK,
	   
	   /* 1: 술 마심; 2: 술 안 마심 */
	   CASE WHEN DRK = 1 THEN 1 ELSE 2
	   END AS DRK
FROM TEMP.DATA
WHERE 2023 - BIRTH >= 45; /* 45세 이상만 */
QUIT;

/*Get dummy*/
DATA TEMP.DATA3;
    SET TEMP.DATA2;

    /* ＲＥＦ: WORK=0 */
    WORK_1 = (WORK = 1);

    /* ＲＥＦ: GENDER=0 */
    GENDER_1 = (GENDER = 1);

    /* ＲＥＦ: EDU=1 */
    EDU_2 = (EDU = 2);
    EDU_3 = (EDU = 3);
    EDU_4 = (EDU = 4);

    /* ＲＥＦ: REGION=1 */
    REGION_2 = (REGION = 2);
    REGION_3 = (REGION = 3);

    /* ＲＥＦ: SCE_STATUS=1 */
    SCE_STATUS_2 = (SCE_STATUS = 2);
    SCE_STATUS_3 = (SCE_STATUS = 3);

    /* ＲＥＦ: HTH_STATUS=1 */
    HTH_STATUS_2 = (HTH_STATUS = 2);
    HTH_STATUS_3 = (HTH_STATUS = 3);

    /* ＲＥＦ: LMT_ATV=0 */
    LMT_ATV_1 = (LMT_ATV = 1);

    /* ＲＥＦ: SMK=1 */
    SMK_2 = (SMK = 2);

    /* ＲＥＦ: DRK=1 */
    DRK_2 = (DRK = 2);

    /* ＤＲＯＰ ＴＨＥ ＶＡＲＩＡＢＬＥＳ */
    DROP WORK GENDER EDU REGION SCE_STATUS HTH_STATUS LMT_ATV SMK DRK;
RUN;
```

## 4. 모델 실행

```swift
PROC REG DATA=TEMP.DATA3;
MODEL HAPPINESS = AGE GENDER_1 EDU_2 EDU_3 EDU_4 
                  REGION_2 REGION_3 SCE_STATUS_2 SCE_STATUS_3 
                  HTH_STATUS_2 HTH_STATUS_3 LMT_ATV_1 
                  SMK_2 DRK_2 WORK_1;
RUN;
```

## 5. 결과

![image 1](https://github.com/user-attachments/assets/91eaf155-9baf-4760-bce4-32c77cd8bf4d)

## 결론
^ㅡ^
