---
layout: single
title: "미로 탈출"
---
미로 탈출
===

문제
---
동빈이는 N x M 크기의 직사각형 형태의 미로에 갇혀 있다. 미로에는 여러 마리의 괴물이 있어 이를 피해 탈출해야 한다. 동빈이의 위치는 (1, 1)이고 미로의 출구는 (N, M)의 위치에 존재하며 한번에 한 칸씩 이동할 수 있다. 이때 괴물이 있는 부분은 0으로, 괴물이 없는 부분은 1로 표시되어 있다. 미로는 반드시 탈출할 수 있는 형태로 제시된다. 이때 동빈이가 탈출하기 위해 움직여야 하는 최소 칸의 개수를 구하시오. 칸을 셀 때는 시작 칸과 마지막 칸을 모두 포함해서 계산한다.

입력 조건
---
첫째 줄에 두 정수 N, M(4 <= N, M <= 200)이 주어집니다. 다음 N개의 줄에는 각각 M개의 정수(0 또는 1)로 미로의 정보가 주어진다. 각각의 수들은 공백 없이 붙어서 입력으로 제시된다. 또한 시작 칸과 마지막 칸은 항상 1이다.

출력 조건
---
첫째 줄에 최소 이동 칸의 개수를 출력한다.


접근
---
- MazeInfo 라는 객체를 만들어 Priority Queue 에 담아 남쪽 우선순위로 검색하게 하였다.
- DFS를 사용하기 보단 BFS 를 사용했다.(하나 하나 탐색하는게 더 효율적인거 같아서)
- 기존 배열에 한칸씩 이동때 마다 이동전 좌표에 1씩을 더 해서 기록했다.


주의
---
- BFS 특징상 무한루프에 빠질 수 있으므로 방문처리를 잘 해둬야 한다.


풀이
---
```java
  static void _5_2() {
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

        int x = 0;
        int y = 0;
        visited[x][y] = true;

        PriorityQueue<MazeInfo> q = new PriorityQueue<>();
        q.offer(new MazeInfo(x, y));

        while (!q.isEmpty()) {

            MazeInfo now = q.poll();
            x = now.x;
            y = now.y;

            for (int i = 0; i < 4; i++) {
                int nx = x + dx[i];
                int ny = y + dy[i];

                if (0 <= nx && nx < n && 0 <= ny && ny < m && map[nx][ny] == 1 && !visited[nx][ny]) {
                    visited[nx][ny] = true;
                    map[nx][ny] = map[x][y] + 1;
                    q.offer(new MazeInfo(nx, ny));
                }
            }
        }
        System.out.println(map[n-1][m-1]);
}


class MazeInfo implements Comparable<MazeInfo> {

    int x;
    int y;

    public MazeInfo(int x, int y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public int compareTo(MazeInfo o) {
        if (x == o.x) {
            return o.y - this.y;
        }
        return o.x - this.x;
    }
}
```

헤맨 부분
---
이동칸수를 얻을때 count 라는 변수를 만들어 더하는 식으로 하려 했으나 <br>
그렇게 되면 이동시 되돌아오는 칸수까지 더 해지게 되서 문제를 못풀게된다.<br>
그래서 한칸씩 이동하는걸 배열 자체에 기록해서 그걸 가지고 답을 구했다.<br>
DFS 로는 쉽게 풀수 있으나 BFS 로 풀려다 좀 헤깔렷다.<br>
