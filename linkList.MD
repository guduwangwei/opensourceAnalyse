### 链表算法操作

* 合并两个有序链表（升序）是的合并的链表有序

 ![合并](https://github.com/guduwangwei/opensourceAnalyse/blob/master/pic/MergeSortTwoLinkList.png)

````c++
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    ListNode* mergeTwoLists(ListNode* l1, ListNode* l2) {
        ListNode * p = NULL;
        ListNode * h = NULL;
        if(l1 == NULL) return l2;
        if(l2 == NULL) return l1;
        if(l1->val > l2->val){
             h= l2;
            l2 = l2->next;
           
        }else{
              h =l1;
            l1=l1->next;
        }
        p = h;
        while(l1 && l2){
            if(l1->val > l2->val){
                 p->next= l2;
                l2 = l2->next;
            }else{
                p->next  = l1;
                l1 = l1->next;
            }
            p=p->next;
            
        }
        if(l1){
            p->next= l1;
        }else{
            p->next= l2;
        }
        return h;
        
        
    }
};
````



