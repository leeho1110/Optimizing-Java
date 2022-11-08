### 개요

벤저민 J. 에번스의 '자바 최적화'를 읽고 정리한 레포지토리입니다.

---

### 서평

JVM에서 사용중인 가비지 컬렉터의 종류와 내부 동작 원리가 궁금해 읽게 되었습니다. GC 관련 내용을 학습하며, JVM 내부 아키텍쳐에 대한 이해도 필요했었는데요. [Java Performance Fundamental](https://performeister.tistory.com/75) 저자분께서 공유해주신 서적 자료와 여러 페이지들을 통해 학습할 수 있었습니다. 정리된 자료는 [여기](https://github.com/leeho1110/optimizing-Java/blob/main/JVM%20Internal/JVM%20Internal.md)서 확인하실 수 있습니다.

초반에는 성능 최적화를 위한 방법과 모니터링에 대한 이야기들이 나옵니다. 가비지 컬렉션에 대한 본격적인 내용은 6장부터 시작됩니다. 

실무에서 JVM을 직접 다룰 일은 많지 않습니다. JVM 구현체를 만들 일은 아예 없죠. 그럼에도 불구하고 학습해야 하는 이유는 뭘까요? 자신이 사용하는 기술을 제대로 알고 활용하는 것과 아닌 것은 효율성 및 견고함에서 차이가 발생하기 때문입니다.

장시간의 STW로 인해 서버 장애가 발생했다면 어떻게 해결할까요? 서버가 OOM Killer에 의해 계속 죽는다면 어떻게 해결할까요? 장애 해결을 위해 필요한 시야는 자신의 발 밑에 축적되온 지식의 부피에 따라 달라집니다. 

**꾸준하게 기반 지식을 학습하고 이를 활용할 수 있다면, 더 높은 시야에서 문제점을 정의하고 해결**할 수 있습니다. 이는 코드로써 비즈니스를 지탱하는 개발자가 고객의 불편함과 장애 해결에 필요한 시간 비용을 줄일 수 있다는 것을 의미합니다. 이것이 제가 꾸준히 학습하는 이유입니다. 

이번 서적을 통해 GC에 대해 딥다이브할 수 있었습니다. 물론 난이도가 꽤 어려웠습니다. 만약 GC에 대한 내용을 알고 싶지만, [공식 문서](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)를 읽기 부담스러우시다면 읽어보셔도 좋을 듯 합니다.

---

### Q&A

- ***Q. GC는 무엇인가요? 왜 필요할까요?***
    
    GC는 Garbage Collector의 약자로써 **더 이상 사용(참조)되지 않는 객체들을 메모리에서 제거해 Heap Memory를 재사용할 수 있도록 정리해주는 프로그램**입니다. 자바에서는 개발자가 메모리를 직접 핸들링하지 않습니다. 대신 별도의 프로그램이 메모리를 할당하고 해제하는 작업을 수행합니다. 이 때 ‘해제’의 대상이 되는 더 이상 사용되지 않는 메모리 공간을 Garbage라고 부릅니다. 이러한 Garbage는 말 그대로 쓰레기이기 때문에 Heap 메모리 영역의 재사용을 위해선 반드시 제거되어야 합니다. 그래야 다음에 사용할 데이터들을 적재할 수 있으니까요. 
    
    또한 해제,할당 과정에서는 메모리 공간이 정렬되지 않아, 사용 가능한 메모리 공간의 크기가 실제 빈 공간보다 적은 Memory Fragmentation 현상이 나타나는데요. 이런 문제점들도 해결해줍니다. 결국은 **효율적인 메모리 재사용을 위해서 존재**하는 것이죠.
    
- ***Q. Garbage Collector에서 Young, Old Memory는 왜 존재할까요? 존재하지 않을수도 있을까요?***
    
    Generationl Algorithm은 Weak Generational Hypothesis, 약한 세대 가설에 기반합니다. 새로 새성된 객체 중 아주 짧은 시간만 살아있는 객체가 대부분이고, 만약 살아남는다면 객체의 기대 수명이 매우 길다는 가정을 하는 것이죠. 기대 수명이 짧은 객체들은 빠르게 GC를 통해 메모리 가용성을 높히는 것이 좋습니다. 반면 기대 수명이 긴 객체들은 애플리케이션의 프로세스에 방해가 되지 않도록 효율적으로 처리해야하죠. 객체의 기대 수명에 맞는 작업을 수행할 수 있도록 그 위치를 Young, Survivor, Old 영역으로 나눠 배치하게 된 것입니다. 
    
    물론 Garbage First GC, G1 GC부터는 이러한 메모리의 물리적 구분을 없애고 Region 방식을 사용해 배치하게 됩니다. 물리적 구분만 없을 뿐 Generational Algorithm은 존재합니다.
    
    - ***Q. Generationl Algorithm은 무엇일까요?***
        
        Generational Argorithm은 Heap 영역을 Young, Survivor, Old으로 나눠 GC하는 알고리즘입니다. Heap을 age별로 sub heap으로 나누고 그 중 Youngest generation sub heap을 자주 GC하는 방식이죠. Generationl Algorithm은 각 Sub-heap에서 다른 Mark-and-sweep 알고리즘 등을 함께 사용하는 경우가 많습니다. 이를 통해 Memory Fragmentation같은 문제점을 해결할 수 있습니다.
        
- ***Q. Hotspot JVM 기준으로 어떤 Garbager Collector들이 있을까요? 각 Garbage Collector들의 특징과 장단점은 무엇일까요?***
    
    Hotspot JVM 기준 최소 Serial Collector, Parallel Collector, CMS Collector, Garbage First Collector 등이 존재합니다. 
    
    - Serial Collector는 1개의 CPU를 사용하는 GC로 코어 수가 적은 경우 적합한 선택지일 확률이 높습니다. 필요한 동작이 많은 CMS, G1 GC 대비 가장 간단한 작업으로 GC가 가능합니다. 다만 CPU를 하나만 사용하는 만큼 GC 동작 시 다른 작업을 수행할 수 없으므로 동시성이 떨어집니다.
    - Parallel Collector는 Serial Collector의 동시성 문제를 해결합니다. 멀티스레드 기반으로 동작하며 처리율(Throughput)에 최적화되어 있습니다. 멀티스레드 기반이기에 Server Class의 Default GC이며 CPU가 1개인 경우 동작하지 않습니다.
    - CMS Collector는 Serial, Parallel Collector에서 발생하는 Old 영역의 STW 시간을 최대한 줄이기 위한 테뉴어드(Old) 전용 Garbage Collector입니다. 대신 GC 수행 시 Initial Mark, Remark 때 STW가 반드시 2번 발생합니다. GC 사이클과 동시에 애플리케이션 스레드가 동작하기 때문에 처리율이 일시적으로 감소합니다. 또한 Compaction 과정이 없어 Memory Fragmentation와 같은 문제가 발생할 수도 있습니다. 
        
    - G1 Collector는 CMS Collector를 대체하기 위한 나온 Low-Pause & Concurrent Garbage Collector 입니다. 대용량 메모리를 가지는 멀티프로세싱 머신을 대상으로 하고 있습니다. 지금까지 소개된 Garbage Collector들과 가장 큰 차이점은 객체 할당 방식입니다. 영 세대, 올드 세대가 존재하긴 하지만 세대별로 연속적으로 메모리에 배치되지는 않습니다. 이전처럼 세대가 아닌 Region으로 구성됩니다. 메모리 수집 시에는 Region들에 대한 메모리의 할당률을 SATB를 통해 파악하고 있다가 효율적으로 메모리를 확보할 수 있는 것들을 **우선적으로** 수집합니다
