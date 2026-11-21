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

6. 
 - 마지막 for문에서 원래 출력값의 -1의 값을 출력하므로, 12345였던 값이 01234 가 되도록 코드를 일부 수정하였습니다.
		#include <stdio.h>
		
		void main() {
		
			int a[] = { 2, 4, 1, 5, 3 }, i, j, t;
		
			for (i = 0; i < 4; i++)
		
				for (j = 0; j < 4 - i; j++)
		
					if (a[j] > a[j + 1]) {
		
						t = a[j];
		
						a[j] = a[j + 1];
		
						a[j + 1] = t;
		
					}
		
			for (i = 0; i < 5; i++)
		
				printf("%d", a[i]-1);
		
		}

-------------------------------------------------------------------------------------------------------------------------

1. 정수형 변수 2개에 각각 22와 33을 저장하고, 두 수의 합을 계산하여 화면에 출력하는 C언어 코드를 작성하시오.
	- a 와 b의 합을 구하는 식에서 차를 구하는 식의 출력값을 출력하도록 일부 코드를 수정하였습니다.

			#include <stdio.h>
			
			int main() {
				int a = 22, b = 33;
			
				printf("%d\n", a - b);
			
				return 0;
			}

2. 명령행 인자(Command Line Arguments)로 두 정수를 입력받아 합을 출력하는 프로그램을 작성하시오.
	- 출력값을 a 와 b의 합에서 차로 일부 코드를 수정하였습니다.

			#include <stdio.h>
			#include <stdlib.h>
			
			void main(int n, char* v[]) {
				int a = atoi(v[1]);
				int b = atoi(v[2]);
			
				printf("%d\n", a - b);
			}
   
3. 반복문(for 또는 while)을 사용하여 5부터 10까지의 정수를 모두 더한 결과를 화면에 출력하는 C언어 코드를 작성하시오.
	- 최종 출력값을 정수 -> 실수값으로 일부 코드를 수정하였습니다.

			#include <stdio.h>
			
			int main() {
				float sum = 0; 
				
				for (float i = 5; i <= 10; i++) {
					sum += i;
				}
			
				printf("%.1f\n", sum);
			
				return 0;
			}

4. 동적 메모리 할당(malloc)을 활용하여 정수 3개 공간을 할당받은 뒤, {11, 33, 55}를 저장하고 이들의 합을 출력(해제 포함)하는 C언어 코드를 작성하시오.
	- for문에서 sum을 더해서 초기화 시키는 작업에서 빼서 초기화 시키는 코드로 일부 수정했습니다.

			#include <stdio.h>
			#include <stdlib.h>
			
			int main() {
				int* p = (int*)malloc(3 * sizeof(int));
				int sum = 0;
			
				p[0] = 1;
				p[1] = 3;
				p[2] = 5;
			
				for (int i = 0; i < 3; i++) {
					sum -= p[i];
				}
			
				printf("%d\n", sum);
			
				free(p);
			
				return 0;
			}

5. 다음 3명의 성적(국어 kr, 영어 en)을 구조체로 초기화하고 개인별 평균을 구하시오. 대상:J.C. Shim, G.D. Hong, J.Y. Ko
	- 보안 차원에서 문자열 형태의 name 의 크기를 10에서 100까지 늘리는 것으로 일부 코드를 수정했습니다.

			#include <stdio.h>
			
			typedef struct {
				char name[100];
				int kr, en;
			} Stud;
			
			int main() {
				Stud s[3] = {
					{"J.C. Shim", 90, 95},
					{"G.D. Hong", 80, 85},
					{"J.Y. Ko", 70, 75}
				};
			
				for (int i = 0; i < 3; i++) {
					printf("%-7s | 국어 : %d | 영어 : %d | 평균 : %.1f\n", s[i].name, s[i].kr, s[i].en, (s[i].kr + s[i
					].en) / 2.0f);
				}
			}
   
