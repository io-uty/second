# 구현

# 📍DESIGN


## [step1 : 구조 세우기]

먼저 kernel/proc.c 파일 내 procdump() 함수를 참고하기 위해 cmd+F procdump()로 해당 함수를 살펴보았다.

```c
void
procdump(void)
{
  static char *states[] = {
  [UNUSED]    "unused",
  [USED]      "used",
  [SLEEPING]  "sleep ",
  [RUNNABLE]  "runble",
  [RUNNING]   "run   ",
  [ZOMBIE]    "zombie"
  };
  struct proc *p;
  char *state;

  printf("\n");
  for(p = proc; p < &proc[NPROC]; p++){
    if(p->state == UNUSED)
      continue;
    if(p->state >= 0 && p->state < NELEM(states) && states[p->state])
      state = states[p->state];
    else
      state = "???";
    printf("%d %s %s", p->pid, state, p->name);
    printf("\n");
  }
}
```

해당 함수는 states라는 포인터배열에 unused, used, sleeping, runnable(runble), running(run), zombie 라는 상태를 숫자가 아닌 문자열 형태로 저장한다.

proc이라는 구조체 *p 와 *state를 정의하였고 proc 시작부터 끝까지 usused상태라면 건너뛰고 위의 상태 중 하나에 해당한다면 state 변수에 상태를 저장한다. 그게 아니라면 “???”를 출력하고 건너뛴다.

-> 본 과제와 상이한 점은 

1. pid에 따라서 pid가 0이라면 모든 수 출력, 아니면 해당하는 pid 의 상태들에 대해 출력
2. procdump에서는 숫자가 존재하지 않으면 “???”를 출력하지만 본과제에서는 아무것도 출력하지 않음
3. 출력 형식(공백 없이 콤마로 이루어진 출력)
   이라고 할 수 있다.
   

## [step2 : 정보 수집]

### #1


*p가 struct proc 형태이기에, 더 깊이 이해해보기 위해 proc구조체를 살펴보았다. 
(cmd+shift+F : "struct proc {" 를 검색) 
  → kernel/proc.h 파일 내에 구현된 것을 알 수 있었다.

  
```c
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // wait_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```


pid, name (최대문자열16), state라는 변수들을 가지고 있는 것을 알 수 있다.


### #2


다음으로 cmd+shift+F : procstate 를 검색
→ kernel/proc.h 파일 내 구현된 procstate 를 찾았고 본 과제를 해결하기 위해 필요한 state들이 정의되어있음을 알 수 있다.

```c
enum procstate { UNUSED, USED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };
```



### #3 : NPROC, NELEM


procdump()함수에서 NPROC, NELEM 이 처음 보는 것들이라 해당 단어가 정의된 파일을 찾아보고 AI를 활용하여 정보를 얻었다.

**[NELEM]**
- kernel/defs.h에 정의되어있다.
- 배열의 원소 개수를 구하는 매크로이자 states 배열이 몇 개짜리인지 자동으로 계산해준다.
  
**[NPROC]**
- cmd+shift+F 검색을 하여 kernel/param.h에 정의되어있다.
- 최대 프로세스 수를 정의해준다.

=> 결론적으로 procdump() 함수에서 사용한대로 동일하게 구현하여도 되겠다는 결론이 나왔다.



## [step3 : 구현방식 구상단계]

유저 프로그램
-(ps 호출)->
라이브러리(usys)
-(ecall 발생)->
커널(syscall dispatcher)
->
sys_ps (wrapper)
->
ps (proc.c 내 진짜 로직)
->
결과 반환


위의 방식으로 systemcall이 이루어진다.



### [Kernel]


### 1. implement a function


kernel/proc.c : 새로운 function 추가

함수 `int ps(int pid)`구현


### 2. Declare the function


kernel/defs.h : 다른 c 파일에서 함수 보이도록

다른 소스에서 참조 가능하도록 `kernel/defs.h`에 함수 선언 추가


### 3. Implement a wrapper func


kernel/sysproc.c : wrapper function 포함


### 4. Register the new system call


kernel/syscall.h

kernel syscall.c 

구현


### [User]


### 5. Declare the function in user.h


user/user.h : 내 프로그램에 시스템콜 보이도록

사용자 프로그램에서 호출 가능하도록 선언


### 6. Add new macro in usys.pl


user/usys.pl : 커널시스템을 call할 방법을 정의해줌

`user/usys.pl`에 시스템 콜 매크로 추가

* 참고로 매크로란?

같은 형태의 어셈블리 코드를 자동 생성하는 템플릿을 말한다.


# 📍IMPLEMENTATION


### kernel/proc.c
<img width="1411" height="682" alt="스크린샷 2026-03-26 오후 1 15 04" src="https://github.com/user-attachments/assets/90a93d65-4626-45c7-bcb8-e4cc484717f9" />
<img width="1422" height="452" alt="스크린샷 2026-03-26 오후 1 15 11" src="https://github.com/user-attachments/assets/9850693c-ef49-4f72-a4c2-3df202ed2b0c" />

procdump 함수를 참고하여 ps 함수를 추가했다.

**[Step1]** 단계에서 분석했던, 본 과제와 상이했던 점을 수정하였다.


1. pid 분기 처리
   
procdump()는 모든 프로세스를 무조건 순회하지만, ps()는 파라미터로 받은 pid 값에 따라 동작이 달라진다.

pid가 0이면 전체 프로세스를 순회하고, 0이 아니면 if(p->pid != pid) continue; 조건을 추가하여 해당 pid와 일치하지 않는 프로세스는 건너뛰도록 구현하였다.


2. 유효하지 않은 state 처리
   
procdump()는 states 배열 범위를 벗어나는 경우 state = "???" 로 출력하지만, ps()에서는 else continue;로 변경하여 해당 프로세스를 아무것도 출력하지 않고 건너뛰도록 수정하였다.


3. 출력 형식 변경
   
procdump()는 printf("%d %s %s", ...) 형태로 공백으로 구분하여 출력하지만, ps()는 printf("%d,%s,%s\n", ...) 형태로 콤마로 구분하도록 변경하였다. 

또한 procdump()의 states 배열에는 "sleep ", "run   " 처럼 공백 패딩이 포함되어 있었는데, ps()에서는 "sleep", "run" 으로 공백을 제거하였다.


### kernel/def.h
<img width="2802" height="488" alt="image" src="https://github.com/user-attachments/assets/5dd1966a-4f3f-4388-b892-799a87de3667" />

다른 c 파일에서 함수가 보이도록, 즉 다른 소스에서 참조가 가능하도록 함수 선언을 추가해주었다.


### kernel/syscall.h
<img width="2814" height="380" alt="image" src="https://github.com/user-attachments/assets/713654b4-d8e1-4d35-9a18-4e8770f2d550" />

CPU는 함수의 이름을 모르기 때문에 숫자로 구분해주어야 하기 때문에 systemcall에 번호 22를 정의해주었다.


### kernel/syscall.c
<img width="1411" height="403" alt="스크린샷 2026-03-26 오후 1 14 33" src="https://github.com/user-attachments/assets/7feac149-e429-460b-80a2-591576f10fe9" />

wrapper function을 system call로 등록하기 위해 배열에 추가해주었다.

### kernel/sysproc.c
<img width="2804" height="676" alt="image" src="https://github.com/user-attachments/assets/f6101c81-9834-4e53-8bba-cdcf87a594c2" />

정수값을 받아와야 하기 때문에 void argint(int n, int *ip); 라는 함수를 사용하였다.

(참고로 해당 함수는 n번째 인자를 정수형(int)으로 가져와 ip에 저장하는 함수이다.)


* 해당 함수를 따로 만드는 이유는 크게 두 가지가 있다.

- 1. user와 kernel의 경계를 보호하기위함
     
     -> user는 kernel함수에 직접 접근할 수 없고 이 과정에서 반드시 "sys_XXX 형태를 거쳐야 한다.
    
- 2. 파라미터를 처리하기 위함
     
     -> 보통 argint(), argaddr()로 값을 꺼낸다.

또한, wrapper는 user의 요청을 받아서 kernel함수로 전달해주는 중간 관리자인 것이라고 이해했다.


### user/user.h
<img width="2830" height="756" alt="image" src="https://github.com/user-attachments/assets/fbc4032f-44e3-4c55-815c-69e9298042f1" />

나의 프로그램 안에서 시스템콜이 보이도록, 즉 사용자 프로그램에서 호출이 가능하도록 선언해주었다.

### user/usys.pl
<img width="2810" height="408" alt="image" src="https://github.com/user-attachments/assets/56475faf-3faf-4ab4-94db-b4fb2ec1a382" />

kernel system을 "ps"로 호출하기 위해 정의해주었다.

### user/ps.c
<img width="2832" height="894" alt="image" src="https://github.com/user-attachments/assets/7fbfebd6-9ae8-4080-b6bf-fd1b9f90534e" />

만약 argc 값이 1이라면, 즉 인자가 0개라면 ps함수에 0을 전달해준다. (모든 pid값을 출력할 수 있도록)

(argc는 운영체제가 이 프로그램을 실행했을 때 전달되는 인수의 개수이다.)

그렇지 않다면 배열에 저장된 char타입 값을 int로 변환해주는 함수 atoi 를 사용하여 ps함수에 전달해준다.


### Makefile
<img width="1415" height="255" alt="스크린샷 2026-03-26 오후 1 13 34" src="https://github.com/user-attachments/assets/778f4e1c-90c8-4551-b499-7c83b2cb8953" />


# 📍RESULTS
<img width="425" height="328" alt="image" src="https://github.com/user-attachments/assets/c8958310-4019-45db-b289-8d656d10b6e0" />

인자 없이 ps만 쳤을 때는 pid값이 모두 출력되고, 원하는 pid 값을 입력하면 해당되는 값들만 출력된다.


# 📍TROUBLE SHOOTING


## 1. 변수 타입 이슈 

**1. 문제 상황**

proc.c 내부에 ps 함수를 구현하는 과정에서 변수 found를 bool타입 함수로 구현했는데 컴파일 에러가 발생했다.

**2. 원인 분석**

xv6는 운영체제 그 자체이기 때문에 stdbool.h와 같은 라이브러리를 사용할 수 없었다.

**3. 해결**

found를 bool 타입이 아닌 int 형으로 바꾸어주었고 true는 1, false는 0이라고 구현해서 해당 이슈를 해결하였다.


## 2. ps.c 헤더파일에 발생한 에러

**1. 문제 상황**

헤더파일 #include “user.h”

#include “types.h” 에서 오류가 발생했다.

**2. 원인 분석**

경로 중 임의의 파일명 (3rdGrade.라고 설정해두었음)에 빨간 줄이 떠서, "."때문에 경로 인식하는 데 문제가 있는 것이라고 생각해 파일 명을 변경하였다.

그러나 이 과정에서 저장을 하지 못했고 이전에 써두었던 코드가 모두 날아갔다.

다행히 레포트를 작성하기 위해 일부 옮겨놓았던 코드가 존재했고, 최근 기억을 토대로 코드를 다시 복구하였다. 

이후 다시 실행했지만 동일한 에러가 발생했다.

에러코드 중 

**int user/user.h:40:36: error: unknown type name 'uint'; did you mean 'int'?** 라는 부분이 있었다. 이 부분을 구글링하고 AI에게 질문하여 두 헤더파일의 위치가 바뀌어야 한다는 것을 알게 되었다.

**3. 해결**

types.h에 uint가 정의되어 있고 user.h에서 uint를 사용하기 때문에 types.h 다음에 user.h 가 와야 한다는 것을 알게 되었고 순서를 바꾸고 나서 에러를 해결할 수 있었다.

## 3. sysproc.c에서 함수 구현

argint와 같이, parameter를 받아오는 wrapper함수는 처음 사용해봐서 낯설었다. 

void argint(int n, int *ip); 해당 함수는 n번째 인자를 정수형으로 가져와 ip에 저장하는 기능을 한다.

처음에는 낯설기도 하고 정확히 어떤 역할을 하는지 몰라서 구현 자체를 하지 못했고 구글링을 하며 제공되는 함수들이 어떤 게 있는지와 어떤 역할을 하는지 더 깊게 공부한 끝에 정확히 이해하고 구현할 수 있었다.

## 4. 구상 단계에서의 의문

**1. 문제 상황**

ps, ps 1, ps 2 3 처럼 인자 개수가 달라도 동일한 main() 함수가 호출되는데, 어떻게 처리되는지 이해가 안됐다.

**2. 원인 분석**

int main()으로만 알고 있었고, argc/argv의 존재를 인지하지 못했다.

즉, 가변 인자처럼 개수를 별도로 넘겨야 하는 줄 알았다.

**3. 해결**

int main(int argc, char *argv[])에서 argc가 인자 개수를, argv가 실제 값을 담아준다는 것을 알게 되었다.

(os가 명령어를 파싱한 후 자동으로 argc/argv를 세팅해서 넘겨준다.)

argc를 분기 조건으로 써서 인자 유무에 따라 다른 동작을 구현할 수 있었다.


## 5. argc 개수 판단


**1. 문제 상황**

argc == 0일 때와 아닐 때로 구분해서 구현하였다.

'''c
    if(argc==0)
        ps(0);
''' 

라고 구현했는데, ps(0) 부분에 에러가 떴다. 
        
**2. 원인 분석**

실행 경로가 항상 argv[0]에 들어가기 때문에 무조건 argc 의 개수는 1개 이상이다.

**3. 해결**

조건문에서 기준을 argc==1 로 잡았고, 반복문에서 initial 값도 1로 지정했다.

또한 반복문도 for(int i=0, ...) 이 아닌 for(int i=1, ...) 이라고 수정하였다.

