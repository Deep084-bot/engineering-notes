Today I learnt about some of the basic cryptography algorithms ie Cesur Cipher and Affine Cipher.
Here I am sharing the code
cesurCipher.c
#include<stdio.h>
#include <stdlib.h>

int main(){
    int s, n;
    printf("Enter the shift key : ");
    scanf("%d", &s);
    printf("Enter the length of the string : ");
    scanf("%d", &n);
    while (getchar() != '\n'); 
    char *ch;
    ch = (char *)malloc(n * sizeof(char));
    if(ch == NULL){
        printf("Failed to load memory.");
        return 1;
    }
    for(int i = 0; i < n; i++) scanf("%c", &ch[i]);
    printf("Encrypted string : ");
    for(int i = 0; i < n; i++){
        char a = (((ch[i] - 'a') + s) % 26) + 'a';
        printf("%c", a);
    }
    printf("\n");
    char *arr;
    arr = (char *)malloc(n * sizeof(char));
    printf("Enter the string to be decrypted : ");
    for(int i = 0; i < n; i++) scanf("%c", &arr[i]);
    printf("Decrypted string : ");
    for(int i = 0; i < n; i++){
        char a = (((arr[i] - 'a') - s) % 26) + 'a';
        printf("%c", a);
    }
    printf("\n");
    return 0;
}

affine.c
