# 야구 데이터를 활용한 SQL
Kaggle에서 MLB 경기, 타석, 투구 정보를 담고 있는 CSV파일 데이터를 활용하여, 데이터를 모델링하고 간단한 데이터 분석을 실시하였다.
## Data description
`games_staging`  
- g_id: 경기 고유 번호
- home_team: 홈팀 명
- away_team: 원정팀 명
- home_final_score: 홈팀 득점
- away_final_score: 원정팀 득점
- date: 경기 날짜
- venue_name: 경기장 명

`atbats_staging`
- inning: 이닝
- top: 초 or 말
- ab_id: 타석 고유 번호
- batter_id: 타자 고유 번호
- pitcher_id: 투수 고유 번호
- stand: 좌/우 타석(타자)
- p_throws: 좌/우투(투수)
- event: 타석 결과
- o: 아웃카운트

`pitches_staging`
- start_speed: 초속
- end_speed: 종속
- b_count: 볼 카운트
- s_count: 스트라이크 카운트
- outs: 아웃카운트
- pitch_num: 타석 당 투구 수

## Data modeling
각 Staging 테이블에서 필요한 컬럼만을 뽑아서 테이블을 새롭게 생성한 후, 데이터를 로딩을 실시하였다.
- `games`
```sql
create table games (
	g_id int not null primary key,
	home_team varchar(255),
	away_team varchar(255),
	home_final_score int,
	away_final_score int,
	venue_name varchar(255)
)
```
```sql
insert into games (g_id, home_team, away_team, home_final_score, away_final_score, venue_name)
select g_id, home_team, away_team, home_final_score, away_final_score, venue_name
from games_staging
```
- `atbats`
```sql
create table atbats (
	ab_id int not null primary key,
	g_id int,
	stand varchar(10),
	p_throws varchar(10),
	event varchar(255),
	foreign key (g_id) references games (g_id)
)
```
```sql
insert into atbats (ab_id, g_id, stand, p_throws, event)
select ab_id, g_id, stand, p_throws, event
from atbats_staging
```
- `pitches`
```sql
create table pitches (
	p_id int not null auto_increment primary key,
	ab_id int,
	speed float,
	pitch_type varchar(10),
	pitch_num int,
	foreign key (ab_id) references atbats (ab_id)
)
```
```sql
insert into pitches (ab_id, speed, pitch_type, pitch_num)
select ab_id, start_speed, pitch_type, pitch_num
from pitches_staging
```
- **ERD diagram**
<center>
  <img
    src="erd.png"
    width="500"
    height="200"
  />
</center>

## 데이터 분석
### 경기장 별 평균 득점
```sql
select venue_name, round(avg(home_final_score + away_final_score), 2) avg_score
from games
group by venue_name
order by avg_score desc
limit 10;
```
### 결과
|**venue_name**|**avg_score**|
|:---:|:---:|
|London Stadium|25.00|
|Coors Field|12.96|
|Estadio de Beisbol Monterrey|12.75|
|Globe Life Park in Arlington|11.61|
|Oriole Park at Camden Yards|11.00|
|Fenway Park|10.67|
|PNC Park|10.43|
|Target Field|10.32|
|Nationals Park|10.28|
|Angel Stadium|10.07|

이벤트 경기를 위해 축구장을 개조해서 만든 London Stadium에서 매우 많은 득점이 나왔다. 하지만, 표본 수가 적어서 신뢰도가 떨어진다. 표본 수가 충분히 많은 Coors Field는 고산 지대에 위치하여 기압이 상대적으로 낮아 비거리가 길게 나오기 때문에 많은 득점이 나온 것으로 확인된다.

### 경기장 별 패스트볼 구속
```sql
select g.venue_name, round(avg(p.speed), 2) speed
from games g
join atbats a 
on g.g_id = a.g_id 
join (select ab_id, speed
	  from pitches
	  where pitch_type in ('FF', 'FT', 'FC', 'SI')) p
on a.ab_id = p.ab_id
group by venue_name
order by speed desc
limit 10
```
### 결과
|**venue_name**|**speed**|
|:---:|:---:|
|London Stadium|93.77|
|Citi Field|93.57|
|Coors Field|93.1|
|Yankee Stadium|93.06|
|Great American Ball Park|93.02|
|Marlins Park|92.98|
|Minute Maid Park|92.92|
|PNC Park|92.85|
|Globe Life Park in Arlington|92.79|
|Guaranteed Rate Field|92.78|

패스트볼 구속 비교를 위해, 패스트볼 계열인 포심, 투심, 커터, 싱커를 필터링하였다. 이번에도 London Stadium의 구속이 가장 높게 측정되었지만, 표본 수가 적어 신뢰도는 현저히 떨어진다. 패스트볼 구속은 경기장 별로 유의미한 차이를 보이지 않았고, 강속구 투수가 많은 팀의 경기장에서 높게 측정되었을 가능성이 높다.

### 홈런 갯수와 득점 수의 상관관계
```sql
create table correlation (x float not null, y float not null);

insert into correlation (x, y)
select (g.home_final_score + g.away_final_score), a.home_run
from games g
join (select g_id, count(event) home_run
	  from atbats
	  where event = 'Home Run'
	  group by g_id) a
on g.g_id = a.g_id;

select @ax := avg(x), 
       @ay := avg(y), 
       @div := (stddev_samp(x) * stddev_samp(y))
from correlation;

select sum( ( x - @ax ) * (y - @ay) ) / ((count(x) -1) * @div) cor from correlation;
```
### 결과
```
+--------------------+
|                cor |
+--------------------+
|  0.610837301137650 |
+--------------------+
```
상관 계수를 구하기 위한 테이블을 생성한 후, 홈런 갯수와 득점 수 데이터를 삽입한다. 이후 상관계수 식을 통하여, 두 컬럼간의 상관계수를 구한다. 홈런과 득점은 약 0.61의 상당히 높은 상관계수를 가진다. 즉, 홈런은 승리에 큰 영향을 미친다. 
