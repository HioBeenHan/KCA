# 재미로 만들어보는 카카오톡 채팅방 분석 스크립트

1st commit: 2021-08-31 12:30  
2nd commit: 2021-09-01 01:06


### 1. 환경 및 경로 설정


```python
import re
import pandas as pd; import datetime as dt; import numpy as np
import matplotlib.pyplot as plt; plt.style.use('ggplot')
import locale; locale.setlocale(locale.LC_ALL, 'ko_KR.UTF-8')
import os
from glob import glob
datadir = './'
filelist = glob(datadir+'*.txt');filelist.sort();

# File selection
filename = filelist[-1]
print('Selected file -> :'+filename)
```

    Selected file -> :.\sample.txt
    

### 2. 텍스트 라인 포맷에 맞게 잘라내는 함수 정의


```python
def parse_line( txt ):
  txt_split = txt.split(',')
  time_ = txt_split[0]+','+txt_split[1]
  id_ = txt_split[2].split(':')[0][1:-1].split('/')[0]
  contents_ = txt_split[2].split(':')[1][1:-1]
  contents_ = re.compile('[ㄱ-ㅎ|ㅏ-ㅣ]').sub('',contents_)
  return time_, id_, contents_
```

### 3. chat_data 폴더에 있는 모든 텍스트파일 긁어와서 list에 넣기 


```python
lines, ids, times, institutes = [], [], [], []
for fileIdx in range(len(filelist)):
    #print('Opening .... Filename : [%s]'%datadir+filelist[fileIdx])
    chat = open(datadir+filelist[fileIdx], 'r', encoding="utf-8").readlines()[8:]
    for lineIdx in range(len(chat)):
        line = chat[lineIdx]
        if line is not '\n':
            try:
                time_, id_, contents_ = parse_line(line)
                ids.append(id_); 
                lines.append(contents_)
                times.append(time_); 
            except:
                pass
        
# Save option
saveOpt = True
if saveOpt:
    df = pd.DataFrame( {'timestamp':times, 'nickname':ids, "contents":lines })
    writer = pd.ExcelWriter('chat_data.xlsx', engine='xlsxwriter')
    df.to_excel(writer, sheet_name='Sheet1')
    writer.close()
```

### 4. 이름 별 채팅 횟수 계산 / Rank 나타내기


```python
nicknames = df['nickname'].to_list()

# Count
from collections import Counter
count = dict( Counter(nicknames) )
names = list(count.keys())
counts = np.array(list(count.values()))

# Sort & formulate x, y data
rank = counts.argsort()
names_by_rank, counts_by_rank = [],[]
for rankIdx in range(len(rank)-1,-1,-1):
    # 이름 공개를 피하기 위해 앞 2글자만 쓰기( [:2] )
    print('[Rank %02d] %s: %d'%(len(rank)-rankIdx,names[rank[rankIdx]][:2], counts[rank[rankIdx]]))
    names_by_rank.append(names[rank[rankIdx]][:2]); counts_by_rank.append(counts[rank[rankIdx]])
    #print('[Rank %02d] %s: %d'%(len(rank)-rankIdx,names[rank[rankIdx]], counts[rank[rankIdx]]))
    #names_by_rank.append(names[rank[rankIdx]]); counts_by_rank.append(counts[rank[rankIdx]])

# Visualization
plt.figure(figsize=(5,5))
plt.barh(names_by_rank, counts_by_rank)
ax = plt.gca(); ax.invert_yaxis(); #ax.set_xscale('log')
plt.xlabel('Total message count rank', fontsize=15)
ax.tick_params( axis='y', labelsize= 20 )
ax.tick_params( axis='x', labelsize= 15 )

```

    [Rank 01] 이재: 32
    [Rank 02] 효빈: 31
    [Rank 03] 김보: 22
    [Rank 04] 김정: 19
    [Rank 05] 이규: 12
    [Rank 06] 이유: 6
    [Rank 07] 박지: 4
    [Rank 08] 김해: 2
    [Rank 09] 김서: 1
    


![png](output_8_1.png)


### 5. 특정 이름 가진 사람만 분석하기


```python
# 계산 편의를 위해 리스트로 변환
nicknames = df['nickname'].to_list()
timestamps = df['timestamp'].to_list()
contents = df['contents'].to_list()

target_id = '박지'
times = [timestamps[i] for i in range(len(nicknames)) if target_id in nicknames[i]]
lines = [contents[i] for i in range(len(nicknames)) if target_id in nicknames[i]]

print('발화 시점: %s'%times)
print('발화 내용: %s'%lines)


```

    발화 시점: ['Aug 3, 2021 9:45 AM', 'Aug 6, 2021 11:24 AM', 'Aug 12, 2021 2:20 PM', 'Aug 13, 2021 10:58 AM']
    발화 내용: ['넵 괜찮습니다~~', '학교에 제출할 서류가 있어서 오늘은 재택하겠습니다..!', '넵 저도 괜찮습니답', '저도 오늘 재택하겠습니다']
    

### 6. 발화자 별 발화 시점 구하기

발화 시점을 구하기 앞서, 카카오톡 채팅 export 기능에서 제공하는 timestamp string를 정규식으로 변환해줄 필요가 있음.  
이를 위해, string을 받아 숫자(float) 형식으로 된 timestamp를 return해주는 함수를 먼저 구현해야 함.




```python
from time import mktime
from datetime import datetime
from calendar import monthrange

def format_datestr( string_in ):
    
    # Parse input
    md_str, yHM_str = string_in.split(',')[0], string_in.split(',')[1][1:]
    y = int(yHM_str.split(' ')[0])
    m = ['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec'].index(md_str.split(' ')[0])+1
    d = int( md_str.split(' ')[1] )
    H = int( yHM_str.split(' ')[1].split(':')[0] )
    if yHM_str.split(' ')[-1] == 'PM': H += 12
    M = int( yHM_str.split(' ')[1].split(':')[1] )
        
    # Correct day-, month-, or year-passing
    if H>=24: 
        d += 1; H -= 24
    if d > monthrange(y, m)[1]:
        m += 1; d = 1
    if m > 13:
        y += 1; m = 1
    
    # Output arguments
    string_out = '%04d-%02d-%02d %02d:%02d:00'%(y,m,d,H,M)
    abs_time = mktime(datetime.strptime(string_out,'%Y-%m-%d %H:%M:%S').timetuple())
    return string_out, abs_time

#def which_day(y,m,d):return ['MON','TUE','WED','THU','FRI','SAT','SUN'][datetime.date(y,m,d).weekday()]
```

이 함수를 이용해, 아래와 같이 생긴 2차원의 binary matrix를 구현한 뒤 raster plot을 시각화.

참고로 raster_matrix는 아래와 같이 생김

``` python
                <- 시간 축 -> (시간은 분 단위)
발화자 1 - [ 0 0 0 0 1 0 0 0 1 0 1 ... 0 ]
발화자 2 - [ 1 0 0 0 1 0 0 0 0 0 0 ... 0 ]
발화자 3 - [ 0 0 1 0 1 0 0 0 1 0 0 ... 0 ]
발화자 4 - [ 0 0 0 0 0 0 1 0 1 0 1 ... 0 ]
...
```


```python
# Formatting [Timestamp, Nickname] 2D matrix (타이밍 딕셔너리)
when_by_whom = np.zeros( (df.shape[0],2), dtype=int)
for lineIdx in range(df.shape[0]):
    by = names_by_rank.index(df['nickname'][lineIdx][:2])
    _, timestamp = format_datestr( df['timestamp'][lineIdx] )
    if lineIdx==0: timestamp_base = timestamp
    when_by_whom[lineIdx,:] = [(timestamp-timestamp_base)/60, by] # To set its unit to "minute"

# Formatting 2D binary matrix
raster_matrix = np.zeros( (when_by_whom[-1,0]+1,len(names_by_rank)), dtype='bool')
for lineIdx in range(when_by_whom.shape[0]):
    raster_matrix[ when_by_whom[lineIdx,0], when_by_whom[lineIdx,1] ] = True

# Visualization (raster plot)
plt.figure(figsize=(12,5))
for nameIdx in range(raster_matrix.shape[1]):
    x = np.where(raster_matrix[:,nameIdx])[0]
    y = np.zeros(x.shape[0])+nameIdx
    plt.plot(x,y,'o')
ax = plt.gca()
ax.invert_yaxis()
plt.yticks( ticks = range(len(names_by_rank)), labels = names_by_rank )
plt.ylim((-1, len(names_by_rank)))
plt.title('단톡방 닉네임 별 발화 시점')
plt.xlabel('시간 (분)')
plt.ylabel('발화자');

```


![png](output_14_0.png)


# 시간 잘 가네


```python

```
