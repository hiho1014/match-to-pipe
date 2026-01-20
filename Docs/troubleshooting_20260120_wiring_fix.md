# 2026-01-20 배선 오류(불필요한 경로 점등) 해결 보고서

**작성일**: 2026-01-20  
**작성자**: Antigravity  

---

## 1. 📌 문제 상황 및 목표
*   **문제**: 전구와 직접 상관없는 샛길이나 루프(빙글빙글 도는 길)까지 빛이 들어와 화면이 지저분해짐.
*   **목표**: 전구로 에너지를 전달하는 **'진짜 유효한 경로'**만 빛나게 수정.

---

## 2. 🛠 해결 원리 (BFS Tree 알고리즘)

기존에는 "물이 닿기만 하면 다 빛난다"는 방식이었다면, 이제는 **"전구에서 변압기까지 거꾸로 거슬러 올라가며 선택된 길만 빛난다"**는 방식을 사용합니다.

1.  **너비 우선 탐색(BFS)**: 변압기에서 출발해 전구가 어디 있는지 찾으면서, 각 파이프가 누구에게 에너지를 받았는지(부모 정보) 기록합니다.
2.  **역추적(Backtracing)**: 불이 켜진 전구에서 시작해, 기록된 부모 정보를 타고 변압기까지 거꾸로 올라갑니다.
3.  **선택적 점등**: 이 역추적 과정에서 '지나온 길'에 포함된 파이프들만 불을 켭니다.

**결과**: 전구와 연결은 되어있지만 에너지가 흐르지 않는 샛길은 어둡게 유지됩니다.

---

## 3. 📝 변경 코드 (`checkPath` 함수)

```javascript
checkPath() {
    // 1. 모든 파이프 초기화
    for (let y = 0; y < this.rows; y++) {
        for (let x = 0; x < this.cols; x++) {
            this.pipes[y][x].setEnergized(false, []);
        }
    }

    const startKey = `${this.startPoint.x},${this.startPoint.y}`;
    const adj = this.buildAdjacencyMap(); // 연결 상태 맵 생성
    
    // 2. BFS로 최단 경로 트리 생성
    const parent = new Map();
    const visited = new Set([startKey]);
    const queue = [startKey];

    while (queue.length) {
        const u = queue.shift();
        (adj.get(u) || []).forEach(v => {
            if (!visited.has(v)) {
                visited.add(v);
                parent.set(v, u);
                queue.push(v);
            }
        });
    }

    // 3. 전구에서 시작점으로 역추적하며 색상 할당
    const pipeColors = new Map();
    this.endPoints.forEach(ep => {
        const bulbKey = `${ep.x},${ep.y}`;
        if (visited.has(bulbKey)) {
            this.updateMarker(`bulb_${ep.id}`, true, ep.color);
            let curr = bulbKey;
            while (curr) {
                if (!pipeColors.has(curr)) pipeColors.set(curr, new Set());
                pipeColors.get(curr).add(ep.color);
                if (curr === startKey) break;
                curr = parent.get(curr);
            }
        } else {
            this.updateMarker(`bulb_${ep.id}`, false);
        }
    });

    // 4. 결과 적용
    for (const [key, colors] of pipeColors) {
        const [px, py] = key.split(',').map(Number);
        this.pipes[py][px].setEnergized(true, Array.from(colors));
    }
}
```

---

## 4. ✅ 테스트 결과
*   **성공**: 전구로 가는 길 외에 일부러 연결한 루프와 샛길에 불이 들어오지 않음.
*   **성공**: 여러 전구가 켜질 때 색상이 겹치는 구간도 정상적으로 표현됨.
*   **성공**: 1~5단계 모든 스테이지에서 의도한 대로 동작함.
