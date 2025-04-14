# [고급보건통계학] GEE 예제
### 이경민, 보건행정학과 통합과정 3학기
<br>


## 주제: 일 하는 사람이 일 하지 않는 사람보다 행복한가?
<br>


## H1: 일 하는 사람이 일 하지 않는 사람보다 행복할 것이다.
<br>


## 자료원 및 대상:

- 노동패널(KLIPS) 23-26차 자료 (2020-2023년)
    - 종단자료
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

- Generalized Estimating Equation (GEE)
- Linear Regression



## 분석 도구

- SAS 9.4



## SAS 구현:

## 0. 프로세스

![image](https://github.com/user-attachments/assets/46826115-0191-4617-a20b-819630359122)

## 1. 라이브러리 불러오기

```swift
LIBNAME RAW "mypath\RAW";
LIBNAME TEMP "mypath\TEMP";
```

## 2. 데이터 불러오기

```swift
************************************ Load Dataset **************************************************
KLIPS Data (2020-2023)

WAVE 23-26

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
%MACRO LOAD_DATA(START, END);
%DO W = &START. %TO &END.;
	%LET YEAR = %EVAL(2020 + (&W. - &START.));
	PROC SQL;
	CREATE TABLE TEMP.DATA_&YEAR. AS
	SELECT PID AS PID,
	       P&W.0201 AS WORK, PA&W.8141 AS HAPPINESS,
		   P&W.0101 AS GENDER, P&W.0104 AS BIRTH,
		   P&W.0110 AS EDU, P&W.0121 AS REGION, P&W.6615 AS SCE_STATUS,
		   P&W.6101 AS HTH_STATUS,
		   P&W.6106-1 AS LMT_ATV_6, P&W.6107-1 AS LMT_ATV_7, P&W.6108-1 AS LMT_ATV_8, P&W.6109-1 AS LMT_ATV_9,
		   P&W.6158 AS SMK, P&W.6161 AS DRK, &YEAR AS YEAR
	FROM RAW.KLIPS&W.P;
	QUIT;
	/*결측 값 확인*/
	PROC MEANS DATA=TEMP.DATA_&YEAR. N NMISS;
	RUN;
%END;
%MEND;
%LOAD_DATA(23, 26);
```

## 3. 데이터 전처리

```swift
%MACRO DATA_PREPROCESSING(SYEAR, EYEAR);
%DO Y = &SYEAR. %TO &EYEAR.;
	/*Grouping*/
	PROC SQL;
	CREATE TABLE TEMP.DATA_&Y._2 AS
	SELECT PID, HAPPINESS,

	     /* 0: 미취업자; 1: 취업자 */
	     CASE WHEN WORK = 2 THEN 0 ELSE 1 
		   END AS WORK,
		   
		   /* 0: 여자; 1: 남자 */
		   CASE WHEN GENDER = 2 THEN 0 ELSE 1 
		   END AS GENDER,

		   /*연령*/
		   YEAR - BIRTH AS AGE, 
		   
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
		   END AS DRK, 
		YEAR
	FROM TEMP.DATA_&Y.
	WHERE YEAR - BIRTH >= 45; /* 45세 이상만 */
	QUIT;
%END;
%MEND;
%DATA_PREPROCESSING(2020, 2023);


%MACRO UNION(SYEAR, EYEAR);
PROC SQL;
    CREATE TABLE TEMP.DATA AS
    %DO Y = &SYEAR %TO &EYEAR;
        SELECT * FROM TEMP.DATA_&Y._2
        %IF &Y < &EYEAR %THEN UNION ALL
    %END;
   ORDER BY PID, YEAR;
QUIT;
%MEND;
%UNION(2020, 2023);
```

## 4. 모델 실행

```swift
PROC GENMOD DATA=TEMP.DATA DESCENDING;
    CLASS PID YEAR REGION EDU SCE_STATUS HTH_STATUS GENDER / PARAM=REF REF=FIRST;

    MODEL HAPPINESS = GENDER AGE EDU REGION SCE_STATUS HTH_STATUS LMT_ATV SMK DRK WORK / 
    DIST=NORMAL LINK=IDENTITY;

	/*REPEATED SUBJECT=PID /WITHIN=YEAR TYPE=IND;*/
	/*REPEATED SUBJECT=PID /WITHIN=YEAR TYPE=AR(1);*/
	/*REPEATED SUBJECT=PID / TYPE=AR(1);*/
	/*REPEATED SUBJECT=PID / TYPE=EXCH;*/
	REPEATED SUBJECT=PID /WITHIN=YEAR TYPE=EXCH CORRW;
RUN;
```

## 5. 결과
![image](https://github.com/user-attachments/assets/ac0e535d-6c3d-422d-805d-e8748312b37c)
![image](https://github.com/user-attachments/assets/b2f89b18-63a1-4f75-a717-8382a9716628)
![image](https://github.com/user-attachments/assets/f93c976e-a813-4daf-9c33-3ca24a9b9ec6)
![image](https://github.com/user-attachments/assets/ecd846a5-5c25-4f2d-850b-0ec90813a6e0)


## 결론
^ㅡ^
