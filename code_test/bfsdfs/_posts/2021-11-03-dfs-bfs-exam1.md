---
layout: single
title: "음료수 얼려먹기"
---

문제
---
N × M 크기의 얼음 틀이 있다. 구멍이 뚫려 있는 부분은 0, 칸막이가 존재하는 부분은 1로 표시된다.
구멍이 뚫려 있는 부분끼리 상, 하, 좌, 우로 붙어 있는 경우 서로 연결되어 있는 것으로 간주한다.
이때 얼음 틀의 모양이 주어졌을 때 생성되는 총 아이스크림의 개수를 구하는 프로그램을 작성하라.
다음의 4 × 5 얼음 틀 예시에서는 아이스크림이 총 3개가 생성된다

{% include image.html id="19VuDfkHwBXWoNvCyvLzqyXAT5qylzCGb"%}

입력
---
첫 번째 줄에 얼음 틀의 세로 길이 N과 가로 길이 M이 주어진다. (1 <= N, M <= 1,000)
두 번째 줄부터 N + 1 번째 줄까지 얼음 틀의 형태가 주어진다.
이때 구멍이 뚫려있는 부분은 0, 그렇지 않은 부분은 1이다.

출력
---
한 번에 만들 수 있는 아이스크림의 개수를 출력한다.

입력 예시 1
```
4 5
00110
00011
11111
00000
```

출력 예시 1  

`3`

입력 예시 2
```
15 14
00000111100000
11111101111110
11011101101110
11011101100000
11011111111111
11011111111100
11000000011111
01111111111111
00000000011111
01111111111000
00011111111000
00000001111000
11111111110011
11100011111111
11100011111111
```

출력 예시2  

`8` 

접근
---
- DFS (재귀함수 이용) 를 사용해 얼음이 가능한 부분을 끝까지 검사
- 더이상 얼음이 될부분이 없으면 아이스크림이 완료된것으로 간주하고 count 추가 
- for문을 통해 아직 얼음이 아닌부분을 찾아 검사


주의 
---
- DFS 특징상 방문한 부분을 처리해주지 않으면 무한루프에 빠질수 있음


풀이
---

```java
 static void _5_1() {
        n = sc.nextInt();
        m = sc.nextInt();

        map = new int[n][m];
        visited = new boolean[n][m];

        for (int i = 0; i < n; i++) {
            String lines = sc.next();
            for (int j = 0; j < m; j++) {
                map[i][j] = lines.charAt(j) - '0';
            }
        }

        visited[0][0] = true;
        map[0][0] = 1;

        int cnt = 0;

        for (int i = 0; i < n; i++) {
            for (int j = 0; j < m; j++) {
                if (_5_1_dfs(i, j)) {
                    cnt++;
                }
            }
        }
        System.out.println(cnt);
    }

    static boolean _5_1_dfs(int x, int y) {

        boolean result = false;

        for (int i = 0; i < 4; i++) {

            int nx = x + dx[i];
            int ny = y + dy[i];

            if (0 <= nx && nx < n && 0 <= ny && ny < m && !visited[nx][ny] && map[nx][ny] == 0) {
                visited[nx][ny] = true;
                map[nx][ny] = 1;
                _5_1_dfs(nx, ny);
                result = true;
            }
        }
        return result;
    }
```