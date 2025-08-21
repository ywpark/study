# 명령어의 구조와 주소(Address)

---

## 1. 명령어

### 구조

- `연산코드 (operation code) + 오퍼랜드 (operand)` 로 이루어짐

### 연산코드

- CPU 마다 명령어가 다름
- 데이터 전송
    - `MOVE, STORE, LOAD(FETCH), PUSH, POP`
- 산술/논리 연산
    - `ADD, SUBTRACT, MULTIPLY, DIVIDE, INCREMENT, DECREMENT,
     AND, OR, NOT, COMPARE`
- 제어 흐름 변경
    - `JUMP, CONDITIONAL JUMP, HALT, CALL, RETURN`
- 입출력 제어
    - `READ, WRITE, START IO, TEST IO`

### 오퍼랜드

- 연산에 사용되는 데이터
- 연산에 사용되는 데이터가 저장된 위치 ( = 주소필드 )
    - 오퍼랜드의 데이터크기의 한계 때문에 직접적인 값을 다루지 않고 주소필드를 활용
- 오퍼랜드는 없는 경우도 있고 한개 이상인 경우도 존재

## 2. 주소 (Address)

### 유효 주소 (Effective Address)

- 연산에 사용할 데이터가 저장된 주소

### 명령어 주소 지정 방식 (Addressing Modes)

- 연산에 사용할 데이터가 저장 된 위치를 찾는 방법
    - 즉, 유효 주소를 찾는 방법
- 주소 지정 방식
    - `즉시 주소 지정 방식 (Immediate Addressing Mode)`
        - 가장 간단한 형태의 주소 지정 방식
        - 연산에 사용할 데이터를 오퍼랜드 필드에 직접 명시
        - 연산에 사용할 데이터의 크기가 작아질 수 있지만 빠름
    - `직접 주소 지정 방식 (Direct Addressing Mode)`
        - 오퍼랜드 필드에 유효 주소를 직접적으로 명시
        - 유효 주소를 표현할 수 있는 크기가 연산 코드만큼 줄어듦
    - `간접 주소 지정 방식 (Indirect Addressing Mode)`
        - 오퍼랜드 필드에 유효 주소의 주소를 명시
        - 다른 주소 지정 방식에 비해 속도가 느림
    - `레지스터 주소 지정 방식 (Register Addressing Mode)`
        - 연산에 사용할 데이터가 저장된 레지스터 명시
        - 메모리에 접근하는 속도보다 레지스터에 접근하는 것이 빠름
    - `레지스터 간접 주소 지정 방식 (Register Indirect Addressing Mode)`
        - 연산에 사용할 데이터를 메모리에 저장
        - 메모리 주소를 저장한 레지스터를 오퍼랜드 필드에 명시