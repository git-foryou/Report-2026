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
