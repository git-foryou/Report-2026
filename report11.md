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

3. 
 - 3행 4열 크기의 행렬 중 3행 4열에 있는 값을 0으로 바꾼 후 합을 진행했습니다.
       #include <stdio.h>
       
       
       
       void main() {
       
       	int a[3][4] = { 1, 2, 3, 4,
                        5, 6, 7, 8,
                       9, 10, 11, 0 };
       
       	int i, j, s = 0;
       
       
       
       	for (i = 0; i < 3; i++)
       
       		for (j = 0; j < 4; j++)
       
       			s += a[i][j];
       
       
       
       	printf("%d\n", s);
       
       }

4.
 - struct 구조체 이름을 S에서 LIST로 바꿔서 일부 코드를 수정하였습니다.
   
       #define _CRT_SECURE_NO_WARNINGS
       
       #include <stdio.h>
       
       
       
       struct LIST {
       
       	int k, e, m;
       
       };
       
       
       
       void main() {
       
       	struct LIST a[2] = { {80, 90, 85}, {70, 75, 95} };
       
       	FILE* p = fopen("my.bin", "wb");
       
       	fwrite(a, sizeof(struct LIST), 2, p);
       
       	fclose(p);
       
       }

5. 
 - 출력되는 값의 소수점 자릿수가 2 -> 1번째 자리까지 나타내는 것으로 일부 코드가 수정되었습니다.

		#define _CRT_SECURE_NO_WARNINGS
		
		#include <stdio.h>
		
		 
		
		struct S {
		
		int k, e, m;
		
		};
		
		 
		
		void main() {
		
		struct S a[2];
		
		FILE *p = fopen("my.bin", "rb");
		
		fread(a, sizeof(struct S), 2, p);
		
		fclose(p);
		
		 
		
		printf("개인1: %.1f, 개인2: %.1f\n", (a[0].k + a[0].e + a[0].m) / 3.0, (a[1].k + a[1].e + a[1].m) / 3.0);
		
		printf("과목1: %.1f, 과목2: %.1f, 과목3: %.1f\n", (a[0].k + a[1].k) / 2.0, (a[0].e + a[1].e) / 2.0, (a[0].m + a[1].m) / 2.0);
		
		}
