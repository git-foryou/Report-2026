## 1 

    #include <stdio.h>
    #include <stdlib.h>
    #include <time.h>
    void main() {
    	int i, h[6] = {0,};
    	srand(time(NULL));
    	for (i = 0;i < 800; i++)
    		h[rand() % 6]++;
    	for (i = 0; i < 6; i++) 
    		printf("[%d] = %f\n", i+1, h[i]/800.0);
    }

## 2

    #include <stdio.h>
    struct N {
        int d;
        struct N* l;
        struct N* r;
    };

    void main() {
        struct N a = { 20, NULL, NULL };
        struct N b = { 10, NULL, NULL };
        struct N c = { 30, NULL, NULL };
        a.l = &b;  
        a.r = &c;  
        printf("%d -> 왼쪽: %d, 오른쪽: %d\n", a.d, a.l->d, a.r->d);
    }

## 3

    #include <stdio.h>
    
    struct N {
        int d;
        struct N* l, * l2, * l3, * root;
    };
    
    int main() {
        struct N a = { 20, NULL, NULL, NULL, NULL }, b = { 10, NULL, NULL, NULL, NULL }, c = { 30,NULL, NULL, NULL, NULL }, d = { 25, NULL, NULL, NULL, NULL }, e = { 40, NULL, NULL };
    
        a.l = &b;  
        a.l2 = &c;  
        a.l3 = &d; 
        a.root = &e; 
    
        printf("root : %d -> leaf : %d | leaf_2 : %d -> underLeaf_1 : %d | underLeaf_2 : %d\n", a.d, a.l->d, a.l2->d, a.l3->d, a.root->d);
    
        return 0;
    }
