1.
 - 보안을 강화하기 위해 입력값의 9자리까지만 받아들이기로 수정했습니다.
   
       #define _CRT_SECURE_NO_WARNINGS
       
       #include <stdio.h>
       
       #include <string.h>
       
       
       
       void main(void) {
       
       	char str[100];
       
       
       
       	printf("문장 입력: ");
       
       	scanf("%9[^\n]", str); // 엔터(\n) 전까지 공백 포함 문장 전체 입력
       
       
       
       	// 문자열 끝에서부터 거꾸로 한 글자씩 출력
       
       	for (int i = (int)strlen(str) - 1; i >= 0; i--) {
       
       		printf("%c", str[i]);
       
       	}
       
       }


2.
 - 변수에 저장되있는 정수값을 실수값으로 바꾸었고, 이를 출력하기 위한 코드를 일부 수정했습니다.

       #include <stdio.h>
       
       
       
       int add(float a, float b) {
       
       	return a + b;
       
       }
       
       
       
       void main() {
       
       	float a = 11.3f, b = 22.1f;
       
       	float (*sum)(float, float) = add;
       
       	printf("%.2f\n", sum(a, b));
       
       }
