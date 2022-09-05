# JVM Internals

### Thread

- intro
    - 쓰레드는 프로그램 안에서 실행되는 스레드를 의미합니다. JVM은 동시에 실행되는 다수개의 스레드를 허용하며 OS 스레드와 일대일로 매핑됩니다.
    - Thread local storage, allocation buffers, synchronization ojbects, stackts and program counter와 같은 스레드를 위한 모든 상태가 준비되면 OS 스레드가 생성됩니다. 이 후 자바 스레드가 종료되면 OS 스레드가 회수됩니다.
    - OS는 모든 스레드를 예약하고 사용가능한 CPU에 이들을 디스패치하는 책임을 갖고 있습니다. OS 스레드가 초기화되면 Java 스레드 안의 `run()` 메서드를 호출합니다. run() 메서드가 반환되면 경우 포착되지 않은 예외가 처리되며 OS 스레드는 종료되는 스레드가 마지막 스레드인지 판단해 JVM 종료 여부를 결정합니다(종료되는 스레드가 데몬 스레드가 아닌지를 의미합니다).
    - 스레드가 종료되면 OS 스레드, 자바 스레드 모두 리소스가 해제됩니다.
- JVM System Threads
    - jconsole이나 다른 디버거를 확인하면 JVM 백그라운드에서 수많은 스레드가 동작하고 있는 것을 확인할 수 있습니다. Hospot JVM 내부에는 아래와 같은 스레드들이 동작하고 있습니다.
        - VM thread: 이 스레드는 JVM이 세이프 포인트에 도달한 뒤 수행해야하는 작업(ex. STW 후 진행되는 GC, 스레드 스택 덤프, 스레드 일시 중단 및 biased locking revocation)이 나타날 떄까지 기다립니다. 이러한 작업이 VM 스레드에서 발생해야 하는 이유는 모두 JVM이 힙에 대한 수정이 발생할 수 없는 안전한 지점에 있어야 하기 때문입니다. 짧게는 메모리 오염이 발생하지 않기 위해서입니다.
        - Periodic task thread: 이 스레드는 주기적으로 실행되어야 하는 스케쥴링 작업같은 타이머 이벤트(ex. 인터럽트)에 대한 동작을 수행합니다.
        - GC thread(s): JVM 내부에서 발생하는 다양한 GC 작업을 지원합니다. 위에서 발생하는 STW 상태에서 진행되는 GC 외의 다른 GC를 의미합니다.
        - Compiler thread(s): 런타임에서 바이트 코드를 네이티브 코드로 컴파일합니다.
        - Signal dispatcher thread: JVM 프로세스로 전달된 시그널을 수신하고 적절한 JVM 메소드를 호출해 처리합니다.

---

### Per Thread

- Program Counter(PC)
    - 네이티브 메서드가 아닌 자바 메서드의 현재 명령어(또는 opcode) 주소입니다. 만약 네이티브 메서드인 경우 프로그램 카운터값은 undefiened입니다. 여기서 가리키는 메모리 주소는 Method Aread(~Java 7), Meataspace(Java 8+) 영역을 가리킵니다.
- Stack
    - 각 스레드는 각자만의 고유 스택을 갖습니다. 이 스택에는 호출되는 메서드별로 스택 프레임이 생성되어 쌓이게 됩니다. 프레임은 모든 메서드 호출에 새롭게 생성되어 push되며 정상적인 반환 혹은 메서드 호출 중 예외가 반환될 때 pop됩니다. 스택은 push,pop 외의 별도의 조작 방법은 없기 때문에 힙에 할당될 수 있고 연속적으로 배치될 필요가 없습니다.
- Native Stack
    
    <p align="center"><img src="img/native-method.png"></p>
    
    - 모든 JVM이 네이티브 메서드를 지원하진 않습니다. 하지만 이를 지원하는 경우 Java Native Invocation, JNI를 위한 C-linkage model를 사용하도록 구현되는 경우 네이티브 스택은 C stack이 됩니다. 이 경우 인자의 순서와 반환값은 일반적인 C 프로그램과 동일합니다. 네이티브 메서드는 JVM 구현에 따라 다르지만 일반적으로 JVM 내부로 콜백되고 Java 메서드를 호출합니다.
    - 앞서 자바 메서드가 호출되는 경우 스택 프레임이 생성된다고 말했습니다. 여기서 만약 네이티브 메서드를 수행하는 경우 Native Stack에 새로운 스택 프레임이 생성되어 푸시된 뒤 메서드가 수행됩니다. 수행이 완료되면 네이티브 메서드를 수행한 스택 프레임으로 돌아가지 않고 **새로운 스택 프레임을 생성**해 작업을 수행합니디ㅏ.
    - 이 때 각 스레드별 스택의 크기는 동적일수도 고정될수도 있스빈다. 만약 스레드에 허용된 스택의 크기보다 더 큰 공간이 필요한 경우 StackOverflowError 예외가 발생합니다.
    - 만약 스레드에 새로운 프레임이 필요하지만 만약 이를 할당할 메모리가 없는 경우라면 OutOfMemoryError 예외가 발생합니다.
- Frame
    - 스택에 추가되는 스택 프레임은 내부적으로 아래와 같은 값들을 갖습니다.
        - Local variable array
        - Return value
        - Operand stack
        - Reference to runtime constant pool for class of the current method(현재 메서드가 선언된 클래스에 대한 런타임 상수 풀에 대한 참조)
    1. Local variale array
        - 지역 변수 배열에는 메서드의 수행동안 사용되는 모든 메서드들이 들어갑니다. this에 대한 참조, 메서드 파라미터, 내부에서 선언된 지역변수들이 포함됩니다. 기본 타입들은 실제 값이, 참조 변수들은 객체가 저장된 힙 영역의 참조가 저장됩니다. 만약 메서드가 다른 메서드에 의해 호출된 경우 돌아갈 주소도 저장됩니다.
        - 해당 영역은 0부터 시작되는 0-Base 배열로 구성됩니다. 로컬 메서드, 인스턴스 메서드에는 자동으로 0번 인덱스에 hidden this 레퍼런스가 저장되며, 이를 통해 힙 영역에 있는 클래스의 인스턴스 데이터를 찾아갑니다. 정적 매서드의 경우 힙에 저장되지 않고 Method Area or Metaspace에 저장되기 때문에 hidden this 레퍼런스가 존재하지 않습니다.
        - 64비트의 크기를 갖는 long, double 자료형(2개의 슬롯을 차지)을 제외한 나머지는 하나의 슬롯을 차지합니다.
    2. Operand stack
        - 바이트코드 인스트럭션의 실행동안 일반적인 CPU에서 레지스터가 사용되는 방식과 비슷하게 사용됩니다.
    3. Dynamic Linking
        - 메서드 실행 시 생기는 스택에는 스택 프레임이 쌓이고, 프레임은 런타임 상수 풀에 대한 참조값을 갖고 있습니다. 각 참조값들을 클래스 파일이 보유한 클래스 상수 풀을 가리키며 이는 다이나믹 링킹을 가능하도록 합니다.
        - C/C++ 코드는 일반적으로 개체 파일로 컴파일된 다음 여러 개체 파일이 함께 연결되어 실행 파일이나 dll과 같은 사용 가능한 구조체를 생성합니다. 링킹 단계에서 각 객체 파일의 심볼릭 참조는 최종 실행 파일에 상대적인 실제 메모리 주소로 대체됩니다.
        - 자바 클래스가 컴파일 뒤 갖게 되는 `ldc #3`, `invokevirtual #4` 와 같은 메서드나 변수에 대한 참조는 실제 메모리 주소가 아닌 클래스 상수 풀에 있는 심볼릭 링크입니다. 아래 예시 코드와 바이트코드를 확인하면 실제 메모리 주소 없이 모두 클래스 파일의 상수 풀에 대한 심볼릭 참조로 대체되어 있습니다.
            - 예시 코드와 `javap -v -p -s -sysinfo -constants classes/org/jvminternals/SimpleClass.class` 명령어로 확인한 바이트 코드입니다.
                - 예시 코드
                    
                    ```java
                    package org.jvminternals;
                    
                    public class SimpleClass {
                    
                        public void sayHello() {
                            System.out.println("Hello");
                        }
                    
                    }
                    ```
                    
                - 바이트코드
                    
                    ```java
                    public class org.jvminternals.SimpleClass
                      SourceFile: "SimpleClass.java"
                      minor version: 0
                      major version: 51
                      flags: ACC_PUBLIC, ACC_SUPER
                    Constant pool:
                       #1 = Methodref          #6.#17         //  java/lang/Object."<init>":()V
                       #2 = Fieldref           #18.#19        //  java/lang/System.out:Ljava/io/PrintStream;
                       #3 = String             #20            //  "Hello"
                       #4 = Methodref          #21.#22        //  java/io/PrintStream.println:(Ljava/lang/String;)V
                       #5 = Class              #23            //  org/jvminternals/SimpleClass
                       #6 = Class              #24            //  java/lang/Object
                       #7 = Utf8               <init>
                       #8 = Utf8               ()V
                       #9 = Utf8               Code
                      #10 = Utf8               LineNumberTable
                      #11 = Utf8               LocalVariableTable
                      #12 = Utf8               this
                      #13 = Utf8               Lorg/jvminternals/SimpleClass;
                      #14 = Utf8               sayHello
                      #15 = Utf8               SourceFile
                      #16 = Utf8               SimpleClass.java
                      #17 = NameAndType        #7:#8          //  "<init>":()V
                      #18 = Class              #25            //  java/lang/System
                      #19 = NameAndType        #26:#27        //  out:Ljava/io/PrintStream;
                      #20 = Utf8               Hello
                      #21 = Class              #28            //  java/io/PrintStream
                      #22 = NameAndType        #29:#30        //  println:(Ljava/lang/String;)V
                      #23 = Utf8               org/jvminternals/SimpleClass
                      #24 = Utf8               java/lang/Object
                      #25 = Utf8               java/lang/System
                      #26 = Utf8               out
                      #27 = Utf8               Ljava/io/PrintStream;
                      #28 = Utf8               java/io/PrintStream
                      #29 = Utf8               println
                      #30 = Utf8               (Ljava/lang/String;)V
                    {
                      public org.jvminternals.SimpleClass();
                        Signature: ()V
                        flags: ACC_PUBLIC
                        Code:
                          stack=1, locals=1, args_size=1
                            0: aload_0
                            1: invokespecial #1    // Method java/lang/Object."<init>":()V
                            4: return
                          LineNumberTable:
                            line 3: 0
                          LocalVariableTable:
                            Start  Length  Slot  Name   Signature
                              0      5      0    this   Lorg/jvminternals/SimpleClass;
                    
                      public void sayHello();
                        Signature: ()V
                        flags: ACC_PUBLIC
                        Code:
                          stack=2, locals=1, args_size=1
                            0: getstatic      #2    // Field java/lang/System.out:Ljava/io/PrintStream;
                            3: ldc            #3    // String "Hello"
                            5: invokevirtual  #4    // Method java/io/PrintStream.println:(Ljava/lang/String;)V
                            8: return
                          LineNumberTable:
                            line 6: 0
                            line 7: 8
                          LocalVariableTable:
                            Start  Length  Slot  Name   Signature
                              0      9      0    this   Lorg/jvminternals/SimpleClass;
                    }
                    ```
                    
        - JVM 구현체는 심볼릭 참조를 언제 resolve(실제 메모리 주소로 변환)할지 결정할 수 있습니다.
            - 즉시 분석 혹은 정적 분석(eager resolution or static resolution): 클래스 파일이 로드된 뒤 검증(verified)될 때
            - 지연 분석 (lzay resolution or late resolution): 심볼릭 참조가 실제로 호출되어 사용될 때
        - JVM은 어쨌든 처음 사용될 때 분석(resolution)이 일어난 것처럼 동작해야하며 이 시점에 resolution 에러가 발생합니다.
            - 바인딩 작업은 심볼릭 참조로 식별되는 필드, 메서드, 클래스가 다이렉트 주소로 변환되는 작업을 의미합니다. 이 작업은 심볼릭 링크가 실제 다이렉트 주소로 변환되는 과정 후에는 다시 돌아갈 수 없기 때문에 반드시 딱 한번만 일어나야 합니다.
            - 각 다이렉트 주소들은 변수 혹은 메서드의 런타임 위치와 연결된 저장 구조체에 대한 **오프셋**으로 저장됩니다.

---

### Shared Between Threads

- 힙 영역
- Memory Management
    - JVM에서 메모리는 절대 개발자가 명시적으로 할당 해제되지 않습니다. 대신 가비지 컬렉터가 자동으로 회수합니다.
- Non-Heap(Off Heap) Memory
    - Permanent Generation(~Java 7) 혹은 Metaspace(Java 8+)
        - 이전 PermGen은 JVM에 의해서 크기가 강제되던 영역이었습니다. 하지만 Metatspace는 Native Memory로 OS가 자동으로 혹은 옵션을 통해 크기를 조절하도록 변경됐습니다. 이를 통해 예전 PermGen에서 발생하던 OOM 에러의 가능성이 더 낮아졌습니다.
        - Method area
        - interend strings
        - PermGen과 Metaspace가 저장하는 데이터의 종류
            
            
            |  | Java 7 (PermGen → Heap, Hotspot 관리대상 O) | Java 8 (Metaspace → Native Memory(Off heap), Hotspot 관리대상 X) |
            | --- | --- | --- |
            | Class Metadata | O | O |
            | Method Metadata | O | O |
            | Interend String | O (GC 대상) | Java Heap 영역으로 이동 (GC 대상) |
            | Static Object 변수, 상수 | O | Java Heap 영역으로 이동 (GC 대상) |
    - Code Cache
        - JIT 컴파일러에 의해 네이티브 코드로 변환된 메서드의 저장과 컴파일에 사용되는 곳입니다.
- 클래스별로 Method area에 저장되는 데이터 종류 (Java 7 기준)입니다. 이 영역은 모든 스레드가 공유하는 자원이므로 두 스레드가 만약 로드되지 않은 클래스의 필드나 메서드에 접근하는 경우 반드시 한 번만 로드되어야 합니다.
    - **Classloader Reference**
    - **Run Time Constant Pool**
        - Numeric constants
        - Field references
        - Method References
        - Attributes
    - **Field data**
        - Per field
            - Name
            - Type
            - Modifiers
            - Attributes
    - **Method data**
        - Per method
            - Name
            - Return Type
            - Parameter Types (in order)
            - Modifiers
            - Attributes
    - **Method code**
        - Per method
            - Bytecodes
            - Operand stack size
            - Local variable size
            - Local variable table
            - Exception table
                - Per exception handler
                    - Start point
                    - End point
                    - PC offset for handler code
                    - Constant pool index for exception class being caught