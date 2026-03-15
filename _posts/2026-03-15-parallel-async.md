---
title: CompletableFuture, Virtual Thread, @Async - 병렬 호출
date: 2026-03-15 10:00:00 +/-0800
categories: [Tech Notes, Java]
tags: [java, completablefuture, virtual-thread, async]
---

## CompletableFuture

### 기본 개념

`CompletableFuture`는 Java 8에서 도입된 비동기 프로그래밍 API입니다. 기존 `Future`는 결과를 `get()`으로 블로킹해서 가져와야 했지만, `CompletableFuture`는 **콜백 체이닝**으로 비동기 흐름을 구성할 수 있습니다.

### supplyAsync - 비동기 작업 시작

```java
// 별도 스레드에서 실행하고, 결과를 CompletableFuture로 받는다
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return callExternalApi();  // 이 코드는 다른 스레드에서 실행됨
});
```

`supplyAsync`는 기본적으로 `ForkJoinPool.commonPool()`을 사용합니다. 별도 Executor를 지정할 수도 있습니다.

```java
ExecutorService executor = Executors.newFixedThreadPool(10);

CompletableFuture<String> future = CompletableFuture.supplyAsync(
    () -> callExternalApi(),
    executor  // 사용할 스레드 풀 지정
);
```

### allOf - 여러 작업을 동시에 실행하고 모두 완료될 때까지 대기

```java
CompletableFuture<List<Drug>> easyFuture =
    CompletableFuture.supplyAsync(() -> searchEasyDrug(name));

CompletableFuture<List<Drug>> pillFuture =
    CompletableFuture.supplyAsync(() -> searchPillIdentification(name));

// 두 작업이 모두 완료될 때까지 대기
CompletableFuture.allOf(easyFuture, pillFuture).join();

// 결과 꺼내기
List<Drug> easyResults = easyFuture.join();
List<Drug> pillResults = pillFuture.join();
```

핵심은 **두 API 호출이 동시에 시작**된다는 것입니다. 순차 호출이면 `1초 + 1초 = 2초`이지만, 병렬이면 `max(1초, 1초) = 1초`입니다.

### join vs get

둘 다 결과를 가져오는 메서드이지만 차이가 있습니다.

| 메서드 | checked 예외 | unchecked 예외 |
|--------|-------------|---------------|
| `get()` | `ExecutionException`, `InterruptedException` 처리 필수 | - |
| `join()` | - | `CompletionException`으로 래핑 |

`join()`은 checked 예외를 강제하지 않으므로 스트림이나 람다 안에서 쓰기 편합니다.

```java
// get() - try-catch 필수
try {
    String result = future.get();
} catch (ExecutionException | InterruptedException e) {
    // ...
}

// join() - unchecked 예외
String result = future.join();  // CompletionException이 발생할 수 있음
```

### exceptionally - 예외 발생 시 대체값 반환

```java
CompletableFuture<List<Drug>> future = CompletableFuture
    .supplyAsync(() -> searchEasyDrug(name))
    .exceptionally(ex -> {
        log.warn("약 검색 실패, 빈 리스트 반환: {}", ex.getMessage());
        return Collections.emptyList();  // 에러 시 빈 리스트로 대체
    });
```

외부 API 호출에서 특히 유용합니다. 하나의 API가 실패해도 다른 API 결과는 정상적으로 반환할 수 있습니다.

`exceptionally` vs `handle`의 차이:

```java
// exceptionally: 에러일 때만 실행
.exceptionally(ex -> fallbackValue)

// handle: 성공/실패 모두 실행
.handle((result, ex) -> {
    if (ex != null) return fallbackValue;
    return transform(result);  // 성공 시 변환도 가능
})
```

## Java 21 Virtual Thread

### Platform Thread vs Virtual Thread

기존 Java 스레드(Platform Thread)는 OS 스레드와 1:1로 매핑됩니다. Java의 유저 스레드를 만들면 JNI를 통해 커널 영역을 호출하여 OS가 커널 스레드를 생성하고 매핑합니다. OS 스레드는 비싸서 보통 수백 개가 한계입니다.

```
[Platform Thread 모델]
Java Thread 1  ←→  OS Thread 1  (1:1 매핑, 비쌈)
Java Thread 2  ←→  OS Thread 2
...
Java Thread 200 ←→ OS Thread 200  ← 여기가 한계
```

스레드 1개당 약 1MB 사이즈라고 가정하면, 4GB 메모리 환경에서 최대 4,000개가 한계입니다. 스레드가 많아지면 컨텍스트 스위칭 비용도 기하급수적으로 늘어납니다.

Virtual Thread는 JVM이 관리하는 경량 스레드로, 플랫폼 스레드 위에 **N:M으로 매핑**됩니다.

```
[Virtual Thread 모델]
Virtual Thread 1  ─┐
Virtual Thread 2   ├→ Carrier Thread 1 (Platform Thread)
Virtual Thread 3  ─┘
Virtual Thread 4  ─┐
Virtual Thread 5   ├→ Carrier Thread 2 (Platform Thread)
Virtual Thread 6  ─┘
...
Virtual Thread 10000 → 실제 OS Thread는 수십 개만 사용
```

| 항목 | Platform Thread | Virtual Thread |
|------|----------------|---------------|
| Stack 사이즈 | ~2MB | ~10KB |
| 생성 시간 | ~1ms | ~1us |
| 컨텍스트 스위칭 | ~100us (커널 영역) | ~10us (JVM 영역) |

Virtual Thread는 JVM에 의해 생성되므로 시스템 콜이 필요 없고, 메모리 크기가 Platform Thread의 **1%** 수준입니다.

### Carrier Thread와 동작 원리

Carrier Thread는 Virtual Thread를 실제로 실행하는 **플랫폼 스레드**입니다. Virtual Thread는 carrier thread 위에 "올라타서" 실행됩니다.

Virtual Thread의 기본 스케줄러는 **ForkJoinPool**을 사용합니다. 스케줄러가 carrier thread pool을 관리하고, Virtual Thread의 작업 분배를 담당합니다.

```
[Virtual Thread 내부 구조]
VirtualThread
  ├── carrierThread: 실제 작업을 수행하는 platform thread
  ├── scheduler: ForkJoinPool (carrier thread pool 관리, 작업 스케줄링)
  └── runContinuation: 실제 작업 내용 (Runnable)
```

동작 흐름:

```
1. virtual thread의 runContinuation을 carrier thread의 workQueue에 push
2. ForkJoinPool이 work stealing 방식으로 carrier thread에 작업 분배
3. I/O, Sleep, interrupt 시 workQueue에서 pop → park → 힙 메모리로 이동
4. I/O 완료 시 unpark → 다시 workQueue에 push → carrier thread가 처리
```

```
Virtual Thread A가 실행 중:
  Carrier Thread 1 ← VT-A가 올라타 있음 (mount)

Virtual Thread A가 블로킹 I/O 시작 (외부 API 호출):
  VT-A park → 힙 메모리로 이동 (unmount)
  Carrier Thread 1 ← VT-B가 올라탐 (mount) → 다른 작업 실행!

Virtual Thread A의 I/O 완료:
  VT-A unpark → workQueue에 push
  Carrier Thread 2 ← VT-A가 다시 올라탐 (다른 carrier일 수 있음)
```

핵심은 **블로킹 I/O 중에 carrier thread를 반환**한다는 것입니다. Platform Thread는 I/O 대기 중에도 OS 스레드를 점유하지만, Virtual Thread는 park되어 힙 메모리로 이동하고 carrier thread를 양보합니다. 그래서 외부 API 호출이 많은 상황에서 특히 유리합니다.

### Virtual Thread 라이프사이클 시각화

아래 버튼을 클릭하면 각 단계별로 Virtual Thread가 어떻게 생성되고, mount/unmount되는지 확인할 수 있습니다.

<div style="border:1px solid #ddd; border-radius:12px; padding:20px; margin:20px 0; background:#fafafa;">
<style>
.vt-scene{position:relative;width:100%;overflow:hidden}
.vt-controls{display:flex;gap:8px;align-items:center;flex-wrap:wrap;margin-bottom:16px}
.vt-step-btn{font-size:13px;cursor:pointer;padding:6px 12px;border:1px solid #ddd;border-radius:6px;background:#fff}
.vt-step-btn.active{border-color:#185FA5;color:#185FA5;background:#E6F1FB}
.vt-phase-title{font-size:15px;font-weight:500;margin-bottom:4px}
.vt-phase-desc{font-size:13px;color:#666;line-height:1.6;margin-bottom:16px}
.vt-code-line{font-family:monospace;font-size:12px;background:#f0f0f0;padding:3px 8px;border-radius:4px;display:inline-block;margin:2px 0}
.vt-highlight-rect{transition:opacity .4s}
@keyframes vt-pulse{0%,100%{opacity:.5}50%{opacity:1}}
@keyframes vt-moveDown{0%{transform:translateY(-20px);opacity:0}100%{transform:translateY(0);opacity:1}}
@keyframes vt-moveRight{0%{transform:translateX(-20px);opacity:0}100%{transform:translateX(0);opacity:1}}
@keyframes vt-fadeIn{0%{opacity:0}100%{opacity:1}}
.vt-anim-pulse{animation:vt-pulse 1.2s ease-in-out infinite}
.vt-anim-move-down{animation:vt-moveDown .5s ease-out forwards}
.vt-anim-move-right{animation:vt-moveRight .5s ease-out forwards}
.vt-anim-fade{animation:vt-fadeIn .4s ease-out forwards}
.vt-legend{display:flex;gap:14px;flex-wrap:wrap;margin-top:12px;font-size:12px;color:#666}
.vt-legend-item{display:flex;align-items:center;gap:5px}
.vt-legend-dot{width:10px;height:10px;border-radius:2px}
</style>

<div class="vt-controls" id="vt-ctrls"></div>
<div class="vt-phase-title" id="vt-title"></div>
<div class="vt-phase-desc" id="vt-desc"></div>
<div class="vt-scene">
<svg id="vt-diagram" width="100%" viewBox="0 0 680 520"></svg>
</div>
<div class="vt-legend">
<div class="vt-legend-item"><div class="vt-legend-dot" style="background:#534AB7"></div>Virtual thread / continuation</div>
<div class="vt-legend-item"><div class="vt-legend-dot" style="background:#1D9E75"></div>Carrier thread</div>
<div class="vt-legend-item"><div class="vt-legend-dot" style="background:#D85A30"></div>I/O / park / unpark 이벤트</div>
<div class="vt-legend-item"><div class="vt-legend-dot" style="background:#888780"></div>ForkJoinPool 스케줄러</div>
</div>

<script>
(function(){
const phases=[
{id:1,label:"① runContinuation → workQueue push",title:"1단계: Virtual thread 생성 → runContinuation을 workQueue에 push",
desc:'<span class="vt-code-line">Thread.startVirtualThread(task)</span> 호출 시, JVM은 내부적으로 <span class="vt-code-line">VirtualThread.run()</span>을 실행합니다.<br><br>이 메서드는 task를 <span class="vt-code-line">Continuation</span> 객체로 감싸고, 이를 실행하는 <span class="vt-code-line">runContinuation</span> Runnable을 생성합니다. 이 Runnable은 ForkJoinPool의 기본 스케줄러에 <span class="vt-code-line">submit()</span>되어, 특정 carrier thread의 <b>로컬 workQueue</b> 끝(tail)에 push됩니다.',
svg:drawPhase1},
{id:2,label:"② Work stealing으로 작업 분배",title:"2단계: ForkJoinPool의 work stealing 스케줄링",
desc:'ForkJoinPool은 각 carrier thread마다 독립적인 <b>deque(양방향 큐)</b>를 유지합니다.<br><br>각 carrier thread는 자신의 workQueue에서 <span class="vt-code-line">LIFO(tail에서 pop)</span> 방식으로 작업을 꺼냅니다. 만약 자신의 큐가 비어 있으면, <b>다른 carrier thread의 큐 head에서 FIFO로 훔쳐옵니다(work stealing)</b>.',
svg:drawPhase2},
{id:3,label:"③ I/O → park → 힙 메모리 이동",title:"3단계: I/O 발생 → park → continuation 힙 저장 (unmount)",
desc:'Virtual thread가 blocking I/O를 만나면:<br><br>1. <span class="vt-code-line">VirtualThread.park()</span>가 호출됩니다<br>2. <span class="vt-code-line">Continuation.yield()</span>가 실행되어 현재 실행 상태를 <b>힙 메모리의 StackChunk 객체</b>에 복사합니다<br>3. Carrier thread의 OS 스택에서 virtual thread의 프레임이 <b>완전히 제거(unmount)</b>됩니다<br>4. Carrier thread는 즉시 다음 작업을 처리할 수 있습니다',
svg:drawPhase3},
{id:4,label:"④ I/O 완료 → unpark → 재스케줄링",title:"4단계: I/O 완료 → unpark → workQueue 재진입 (remount)",
desc:'I/O가 완료되면:<br><br>1. Poller 스레드가 <span class="vt-code-line">VirtualThread.unpark()</span>를 호출합니다<br>2. runContinuation이 다시 ForkJoinPool에 submit됩니다<br>3. <b>원래와 다른 carrier thread</b>의 workQueue에 들어갈 수 있습니다<br>4. 힙의 StackChunk에서 스택 프레임이 <b>carrier thread의 OS 스택으로 복원(remount)</b>됩니다',
svg:drawPhase4}
];

let cur=0;
const ctrls=document.getElementById('vt-ctrls');
const titleEl=document.getElementById('vt-title');
const descEl=document.getElementById('vt-desc');
const svg=document.getElementById('vt-diagram');

phases.forEach(function(p,i){
var b=document.createElement('button');
b.className='vt-step-btn'+(i===0?' active':'');
b.textContent=p.label;
b.onclick=function(){goTo(i)};
ctrls.appendChild(b);
});

function goTo(i){
cur=i;
document.querySelectorAll('.vt-step-btn').forEach(function(b,j){b.classList.toggle('active',j===i)});
titleEl.textContent=phases[i].title;
descEl.innerHTML=phases[i].desc;
phases[i].svg();
}

var MK='<marker id="vta" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></marker>';

function baseSvg(){
return '<defs>'+MK+'</defs>'+
'<rect x="30" y="20" width="620" height="130" rx="12" fill="none" stroke="#888780" stroke-width="1" stroke-dasharray="4 3"/>'+
'<text x="50" y="44" font-size="13" font-weight="500" fill="#5F5E5A">ForkJoinPool (default scheduler)</text>'+
carrierSvg(60,70,"Carrier-1","CT1")+carrierSvg(260,70,"Carrier-2","CT2")+carrierSvg(460,70,"Carrier-3","CT3")+
'<rect x="60" y="190" width="160" height="110" rx="8" fill="none" stroke="#0F6E56" stroke-width="0.5" stroke-dasharray="3 2"/>'+
'<text x="140" y="210" text-anchor="middle" font-size="12" fill="#0F6E56">workQueue-1 (deque)</text>'+
'<rect x="260" y="190" width="160" height="110" rx="8" fill="none" stroke="#0F6E56" stroke-width="0.5" stroke-dasharray="3 2"/>'+
'<text x="340" y="210" text-anchor="middle" font-size="12" fill="#0F6E56">workQueue-2 (deque)</text>'+
'<rect x="460" y="190" width="160" height="110" rx="8" fill="none" stroke="#0F6E56" stroke-width="0.5" stroke-dasharray="3 2"/>'+
'<text x="540" y="210" text-anchor="middle" font-size="12" fill="#0F6E56">workQueue-3 (deque)</text>'+
'<rect x="160" y="390" width="360" height="110" rx="14" fill="none" stroke="#888780" stroke-width="1" stroke-dasharray="4 3"/>'+
'<text x="340" y="415" text-anchor="middle" font-size="13" font-weight="500" fill="#5F5E5A">Heap memory</text>';
}

function carrierSvg(x,y,name,id){
return '<g id="'+id+'"><rect x="'+x+'" y="'+y+'" width="160" height="60" rx="8" fill="#E1F5EE" stroke="#0F6E56" stroke-width="0.5"/>'+
'<text x="'+(x+80)+'" y="'+(y+25)+'" text-anchor="middle" font-size="13" font-weight="500" fill="#085041">'+name+'</text>'+
'<text x="'+(x+80)+'" y="'+(y+43)+'" text-anchor="middle" font-size="11" fill="#0F6E56">OS thread</text></g>';
}

function vtSvg(x,y,name,cls){
return '<rect x="'+x+'" y="'+y+'" width="130" height="34" rx="6" fill="#EEEDFE" stroke="#534AB7" stroke-width="0.5" class="'+(cls||'')+'"/>'+
'<text x="'+(x+65)+'" y="'+(y+20)+'" text-anchor="middle" font-size="11" font-weight="500" fill="#3C3489" dominant-baseline="central">'+name+'</text>';
}

function stackChunkSvg(x,y,name,cls){
return '<rect x="'+x+'" y="'+y+'" width="140" height="40" rx="6" fill="#FAECE7" stroke="#D85A30" stroke-width="0.5" class="'+(cls||'')+'"/>'+
'<text x="'+(x+70)+'" y="'+(y+15)+'" text-anchor="middle" font-size="11" font-weight="500" fill="#712B13">'+name+'</text>'+
'<text x="'+(x+70)+'" y="'+(y+30)+'" text-anchor="middle" font-size="10" fill="#993C1D">stack frames + locals</text>';
}

function arrowSvg(x1,y1,x2,y2,color,cls,dashed){
return '<line x1="'+x1+'" y1="'+y1+'" x2="'+x2+'" y2="'+y2+'" stroke="'+(color||'#888780')+'" stroke-width="1.2" marker-end="url(#vta)" '+(dashed?'stroke-dasharray="4 2"':'')+' class="'+(cls||'')+'" fill="none"/>';
}

function labelSvg(x,y,text,color,cls){
return '<text x="'+x+'" y="'+y+'" text-anchor="middle" font-size="11" fill="'+(color||'#5F5E5A')+'" class="'+(cls||'')+'">'+text+'</text>';
}

function drawPhase1(){
svg.innerHTML=baseSvg()+
vtSvg(75,228,"VT-1 (runCont.)","vt-anim-fade")+
arrowSvg(140,262,140,280,"#534AB7","vt-anim-fade")+
labelSvg(140,350,"tail ↑ push","#534AB7","vt-anim-fade")+
'<rect x="200" y="340" width="130" height="30" rx="6" fill="#EEEDFE" stroke="#534AB7" stroke-width="1" class="vt-anim-pulse"/>'+
'<text x="265" y="359" text-anchor="middle" font-size="11" font-weight="500" fill="#3C3489">Thread.start()</text>'+
arrowSvg(265,340,160,265,"#534AB7","vt-anim-move-down",true)+
labelSvg(240,320,"submit()","#534AB7","vt-anim-fade")+
labelSvg(340,260,"(비어있음)","#888780","")+
labelSvg(540,260,"(비어있음)","#888780","");
}

function drawPhase2(){
svg.innerHTML=baseSvg()+
'<rect x="75" y="85" width="130" height="28" rx="5" fill="#EEEDFE" stroke="#534AB7" stroke-width="1" class="vt-anim-fade"/>'+
'<text x="140" y="103" text-anchor="middle" font-size="11" font-weight="500" fill="#3C3489" dominant-baseline="central">VT-1 ▶ RUNNING</text>'+
arrowSvg(140,190,140,135,"#1D9E75","vt-anim-move-down")+
labelSvg(108,175,"pop (LIFO)","#0F6E56","vt-anim-fade")+
'<path d="M460 250 C420 280, 230 280, 260 250" fill="none" stroke="#D85A30" stroke-width="1.2" stroke-dasharray="4 2" marker-end="url(#vta)" class="vt-anim-fade"/>'+
'<text x="360" y="285" text-anchor="middle" font-size="11" fill="#993C1D" class="vt-anim-fade">work stealing (FIFO, head에서)</text>'+
vtSvg(275,228,"VT-2","vt-anim-fade")+
vtSvg(275,265,"VT-3","vt-anim-fade")+
labelSvg(540,260,"(비어있음)","#888780","");
}

function drawPhase3(){
svg.innerHTML=baseSvg()+
'<rect x="75" y="85" width="130" height="28" rx="5" fill="#FAECE7" stroke="#D85A30" stroke-width="1" class="vt-anim-pulse"/>'+
'<text x="140" y="103" text-anchor="middle" font-size="11" font-weight="500" fill="#712B13" dominant-baseline="central">I/O blocked! park()</text>'+
'<path d="M140 130 L140 350 L270 390" fill="none" stroke="#D85A30" stroke-width="1.5" stroke-dasharray="5 3" marker-end="url(#vta)" class="vt-anim-fade"/>'+
'<text x="80" y="270" font-size="11" fill="#993C1D" class="vt-anim-fade">Continuation.yield()</text>'+
'<text x="80" y="285" font-size="11" fill="#993C1D" class="vt-anim-fade">→ 스택 프레임 복사</text>'+
stackChunkSvg(200,430,"VT-1 StackChunk","vt-anim-move-down")+
'<rect x="75" y="85" width="130" height="28" rx="5" fill="#E1F5EE" stroke="#1D9E75" stroke-width="0.5" class="vt-anim-fade" opacity="0.3"/>'+
'<text x="140" y="103" text-anchor="middle" font-size="11" fill="#085041" dominant-baseline="central" class="vt-anim-fade" opacity="0.6">→ 해방! 다음 작업</text>'+
vtSvg(275,228,"VT-2","vt-anim-fade")+
arrowSvg(340,228,340,190,"#1D9E75","vt-anim-fade")+
arrowSvg(340,190,340,135,"#1D9E75","vt-anim-fade")+
labelSvg(390,175,"즉시 다음 VT 처리","#0F6E56","vt-anim-fade")+
stackChunkSvg(380,430,"VT-4 StackChunk","vt-anim-fade");
}

function drawPhase4(){
svg.innerHTML=baseSvg()+
'<rect x="30" y="345" width="110" height="34" rx="6" fill="#FAECE7" stroke="#D85A30" stroke-width="0.5" class="vt-anim-pulse"/>'+
'<text x="85" y="366" text-anchor="middle" font-size="11" font-weight="500" fill="#712B13" dominant-baseline="central">Poller (epoll)</text>'+
arrowSvg(85,345,85,330,"#D85A30","vt-anim-fade")+
labelSvg(85,325,"I/O 완료!","#D85A30","vt-anim-pulse")+
stackChunkSvg(200,430,"VT-1 StackChunk","vt-anim-fade")+
'<path d="M340 430 Q500 380, 540 300" fill="none" stroke="#534AB7" stroke-width="1.5" marker-end="url(#vta)" class="vt-anim-move-right"/>'+
'<text x="480" y="375" font-size="11" fill="#3C3489" class="vt-anim-fade">unpark()</text>'+
'<text x="480" y="390" font-size="11" fill="#3C3489" class="vt-anim-fade">→ 재submit</text>'+
vtSvg(475,228,"VT-1 (재개)","vt-anim-move-right")+
arrowSvg(540,228,540,135,"#1D9E75","vt-anim-move-down")+
'<rect x="475" y="85" width="130" height="28" rx="5" fill="#EEEDFE" stroke="#534AB7" stroke-width="1" class="vt-anim-move-right"/>'+
'<text x="540" y="103" text-anchor="middle" font-size="11" font-weight="500" fill="#3C3489" dominant-baseline="central">VT-1 ▶ RESUMED</text>'+
labelSvg(540,168,"remount: 스택 복원","#0F6E56","vt-anim-fade")+
'<rect x="430" y="155" width="220" height="20" rx="4" fill="#FAEEDA" stroke="#BA7517" stroke-width="0.5" class="vt-anim-fade"/>'+
'<text x="540" y="169" text-anchor="middle" font-size="10" font-weight="500" fill="#854F0B" dominant-baseline="central">⚡ Carrier-1이 아닌 Carrier-3에서 재개!</text>'+
vtSvg(75,228,"VT-5","vt-anim-fade")+
vtSvg(275,228,"VT-6","vt-anim-fade");
}

goTo(0);
})();
</script>
</div>

### JDK 21에서 park/unpark가 가능한 이유

JDK 21에서는 기존 Thread 모델의 park/unpark 로직에 **Virtual Thread 분기를 추가**했습니다.

- **LockSupport**: 현재 스레드가 Virtual Thread인지 확인 후, VT이면 virtual thread 전용 park/unpark 수행
- **NIOSocketImpl.park()**: Socket I/O 시 VT이면 `Poller.poll()`로 virtual thread park 수행 (기존에는 `Net.poll()`로 커널 영역에 요청)
- **Thread.sleep()**: VT이면 virtual thread 기반 park 수행

기존 Thread 방식에서 **코드 수정 없이** Virtual Thread 기반 컨텍스트 스위칭이 가능한 이유입니다.

### Thread.ofVirtual()

```java
// Virtual Thread 생성
Thread vt = Thread.ofVirtual().start(() -> {
    System.out.println("I'm a virtual thread!");
});

// Virtual Thread용 Executor
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
executor.submit(() -> callExternalApi());
```

`newVirtualThreadPerTaskExecutor()`는 작업마다 새 Virtual Thread를 만듭니다. 풀이 아니라 **매번 새로 생성**하지만, Virtual Thread는 매우 가벼워서 문제 없습니다.

### 성능 비교 (우아한형제들 벤치마크)

우아한형제들 기술블로그에서 256MB 힙, Ngrinder를 사용한 테스트 결과입니다.

**I/O Bound 작업** (300ms sleep API 3번 호출):

| 구분 | 결과 |
|------|------|
| Thread vs Virtual Thread | Virtual Thread가 **51% 이상** 성능 향상 |
| Virtual Thread vs Kotlin Coroutine | Virtual Thread가 **37%** 더 좋은 성능 |
| Virtual Thread vs Reactive (WebFlux) | Virtual Thread가 **111%** 더 좋은 성능 |

특히 vuser(동시 사용자)를 250 이상으로 올리면 Platform Thread 서버는 **죽었지만**, Virtual Thread 서버는 **정상 처리**했습니다.

**CPU Bound 작업** (0~3억 합산 3번):

CPU Bound에서는 **오히려 Platform Thread가 더 빨랐습니다**. Virtual Thread가 carrier thread 위에서 동작하므로, 컨텍스트 스위칭이 없는 순수 CPU 작업에서는 Virtual Thread 생성/스케줄링 비용만 추가됩니다.

> "It is more expensive to run a task in a virtual thread than running it in a platform thread." - Oracle

**결론: Virtual Thread는 I/O Bound 작업에 최적화되어 있고, CPU Bound 작업에는 적합하지 않습니다.** 외부 API 호출이 많은 DrugSearchService 같은 경우에 딱 맞는 기술입니다.

### 카카오페이 성능 비교 결과

카카오페이 기술블로그의 벤치마크에 따르면:

| 구분 | Virtual Thread | 코루틴 |
|------|---------------|--------|
| CPU 바운드 작업 (소수 찾기 10만번) | 약 1,910ms | 약 2,300ms |
| 메모리 사용량 | 약 30MB | 약 67MB |

Virtual Thread가 **10~15% 더 빠르고, 메모리도 절반** 수준이었습니다.

### 다른 경량 스레드 모델과의 비교 정리

| 기준 | Virtual Thread | Kotlin Coroutine | Reactive (WebFlux) |
|------|---------------|-----------------|-------------------|
| 코드 변경 | 거의 없음 | suspend 함수 전파 | 전체 리액티브 전환 필요 |
| 함수 색 문제 | 없음 | suspend 전염성 있음 | Mono/Flux 전염성 있음 |
| 디버깅 | 기존과 동일 (스택 트레이스 유지) | 비동기 스택 트레이스 어려움 | 스택 트레이스 유실 |
| 적용 범위 | 애플리케이션 전체 (Executor 교체) | 메서드 단위 선택 적용 | 전체 스택 전환 |
| JDK 버전 | 21+ 필수 | JDK 버전 무관 | JDK 버전 무관 |
| 러닝 커브 | 낮음 | 중간 (Kotlin 학습 필요) | 높음 (Reactor 학습 필요) |

## spring.threads.virtual.enabled=true

Spring Boot 3.2+에서는 이 설정 한 줄로 Virtual Thread를 활성화할 수 있습니다.

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
```

이 설정을 켜면:
- **톰캣의 요청 처리 스레드**가 Virtual Thread로 바뀜
- `@Async` 작업도 Virtual Thread에서 실행됨
- 기존 코드 변경 없이 적용 가능

즉, 모든 HTTP 요청이 Virtual Thread에서 처리되므로, 외부 API 호출로 블로킹되어도 carrier thread를 반환하여 다른 요청을 처리할 수 있습니다.

## @Async + ThreadPoolTaskExecutor vs SimpleAsyncTaskExecutor

### SimpleAsyncTaskExecutor (기본값)

`@EnableAsync`만 선언하고 커스텀 Executor를 등록하지 않으면 `SimpleAsyncTaskExecutor`가 사용됩니다.

```java
@EnableAsync
@SpringBootApplication
public class MyApplication { }

@Async
public CompletableFuture<List<Drug>> searchAsync(String name) {
    return CompletableFuture.completedFuture(searchEasyDrug(name));
}
```

문제는 `SimpleAsyncTaskExecutor`가 **요청마다 새 스레드를 생성**한다는 것입니다. 풀이 아니므로 동시 요청이 폭주하면 스레드가 무한 생성되어 OOM이 발생할 수 있습니다.

### ThreadPoolTaskExecutor (권장)

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);      // 기본 스레드 수
        executor.setMaxPoolSize(20);       // 최대 스레드 수
        executor.setQueueCapacity(100);    // 대기 큐 크기
        executor.setThreadNamePrefix("async-");
        executor.setRejectionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

| 설정 | 의미 |
|------|------|
| `corePoolSize: 10` | 항상 유지되는 스레드 수 |
| `maxPoolSize: 20` | 큐가 가득 차면 최대 20개까지 스레드 증가 |
| `queueCapacity: 100` | core가 다 바쁘면 100개까지 큐에 대기 |
| `CallerRunsPolicy` | 큐도 가득 차면 호출한 스레드가 직접 실행 (요청 유실 방지) |

### 동작 흐름

```
요청 들어옴 → core 스레드 여유 있음? → YES → core 스레드가 처리
                                      → NO  → 큐에 넣기 (100개까지)
                                               → 큐도 가득? → max까지 스레드 추가 (20개)
                                                              → max도 가득? → RejectionPolicy 실행
```

### 비교 정리

| 기준 | SimpleAsyncTaskExecutor | ThreadPoolTaskExecutor |
|------|------------------------|----------------------|
| 스레드 생성 | 매번 새로 생성 | 풀에서 재사용 |
| 스레드 수 제한 | 없음 (무한 생성 가능) | maxPoolSize로 제한 |
| OOM 위험 | 있음 | 없음 (큐 + 제한) |
| 적합한 상황 | 테스트용 | 운영 환경 |

## Structured Concurrency (Java 21 Preview)

### 문제: CompletableFuture의 에러 전파

```java
CompletableFuture<A> f1 = CompletableFuture.supplyAsync(() -> callApi1());
CompletableFuture<B> f2 = CompletableFuture.supplyAsync(() -> callApi2());

// f1이 실패해도 f2는 계속 실행 중... 자원 낭비!
CompletableFuture.allOf(f1, f2).join();
```

`allOf`는 하나가 실패해도 다른 작업을 취소하지 않습니다.

### Structured Concurrency의 해결

```java
// Java 21 Preview - StructuredTaskScope
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Subtask<A> f1 = scope.fork(() -> callApi1());
    Subtask<B> f2 = scope.fork(() -> callApi2());

    scope.join();            // 모두 완료 대기
    scope.throwIfFailed();   // 하나라도 실패하면 예외 (나머지도 자동 취소!)

    A result1 = f1.get();
    B result2 = f2.get();
}
```

핵심 차이:
- **ShutdownOnFailure**: 하나가 실패하면 나머지 작업을 **자동 취소** → 자원 낭비 방지
- **ShutdownOnSuccess**: 하나가 성공하면 나머지를 취소 → "가장 빠른 결과" 패턴
- `try-with-resources`로 scope가 닫히면 모든 subtask가 정리됨

아직 Preview 기능이므로 프로덕션에서는 주의가 필요합니다. 하지만 방향성을 이해해두면 좋습니다.

## Virtual Thread Pinning (synchronized 블록 문제)

### Pinning이란

Virtual Thread가 carrier thread에서 **내려오지 못하고 고정되는 현상**입니다.

정상 동작:
```
VT-A가 I/O 대기 → carrier thread에서 내려옴 (unmount) → 다른 VT가 사용
```

Pinning 발생:
```
VT-A가 synchronized 블록 안에서 I/O 대기 → carrier thread에서 못 내려옴!
→ carrier thread가 점유된 채로 대기 → 다른 VT가 실행 못 함
```

### 왜 synchronized에서 발생하는가

`synchronized`는 **모니터 락**을 사용하는데, 이 락은 특정 OS 스레드에 바인딩됩니다. Virtual Thread가 carrier thread에서 내려오면 락의 소유권이 깨지기 때문에, JVM이 **내려오는 것을 막습니다**.

```java
// Pinning 발생하는 코드
synchronized (lock) {
    restTemplate.getForObject(url, String.class);  // 블로킹 I/O
    // → carrier thread가 점유된 채로 I/O 대기!
}

// 해결: ReentrantLock 사용
private final ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    restTemplate.getForObject(url, String.class);  // 블로킹 I/O
    // → carrier thread에서 정상적으로 unmount 가능
} finally {
    lock.unlock();
}
```

## 참고 자료

- [카카오페이 - 코루틴과 Virtual Thread 비교와 사용](https://tech.kakaopay.com/post/coroutine_virtual_thread_wayne/)
- [우아한형제들 - Java의 미래, Virtual Thread](https://techblog.woowahan.com/15398/)
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [CompletableFuture API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CompletableFuture.html)
