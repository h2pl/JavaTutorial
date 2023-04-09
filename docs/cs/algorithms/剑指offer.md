# 目录

* [数论和数字规律](#数论和数字规律)
  * [从1到n整数中1出现的次数](#从1到n整数中1出现的次数)
  * [排数组排成最小的数](#排数组排成最小的数)
  * [丑数](#丑数)
* [数组和矩阵](#数组和矩阵)
  * [二维数组的查找](#二维数组的查找)
  * [顺时针打印矩阵。](#顺时针打印矩阵。)
  * [调整数组中数字的顺序，使正数在负数的前面](#调整数组中数字的顺序，使正数在负数的前面)
  * [数组中出现次数超过一半的数字](#数组中出现次数超过一半的数字)
  * [找出前k小的数](#找出前k小的数)
  * [连续子数组的最大和](#连续子数组的最大和)
  * [逆序对](#逆序对)
  * [数字在排序数组中出现的次数](#数字在排序数组中出现的次数)
  * [和为s的两个整数，和为s的连续正数序列](#和为s的两个整数，和为s的连续正数序列)
  * [n个色子的点数](#n个色子的点数)
  * [扑克牌的顺子](#扑克牌的顺子)
  * [数组中重复的数字](#数组中重复的数字)
  * [数组中重复的数字](#数组中重复的数字-1)
  * [构建乘积数组](#构建乘积数组)
  * [数据流的中位数](#数据流的中位数)
  * [滑动窗口中的最大值](#滑动窗口中的最大值)
* [字符串](#字符串)
  * [字符串的排列](#字符串的排列)
  * [替换空格](#替换空格)
  * [第一次只出现一次的字符](#第一次只出现一次的字符)
  * [翻转单词顺序和左旋转字符串](#翻转单词顺序和左旋转字符串)
  * [把字符串转换为整数](#把字符串转换为整数)
  * [表示数值的字符串](#表示数值的字符串)
  * [字符流中第一个不重复的字符](#字符流中第一个不重复的字符)
* [链表](#链表)
  * [从尾到头打印链表](#从尾到头打印链表)
  * [链表倒数第k个节点](#链表倒数第k个节点)
  * [反转链表](#反转链表)
  * [合并两个排序链表](#合并两个排序链表)
  * [复杂链表的复制](#复杂链表的复制)
  * [两个链表的第一个公共节点](#两个链表的第一个公共节点)
  * [孩子们的游戏(圆圈中最后剩下的数)](#孩子们的游戏圆圈中最后剩下的数)
  * [链表的环的入口结点](#链表的环的入口结点)
  * [删除链表中重复的节点](#删除链表中重复的节点)
      * [](#)
  * [二叉搜索树转换为双向链表](#二叉搜索树转换为双向链表)
  * [重建二叉树](#重建二叉树)
  * [树的子结构](#树的子结构)
  * [镜像二叉树](#镜像二叉树)
  * [树的层次遍历](#树的层次遍历)
  * [二叉树的深度](#二叉树的深度)
  * [判断是否平衡二叉树](#判断是否平衡二叉树)
  * [二叉搜索树的后序遍历](#二叉搜索树的后序遍历)
  * [二叉树中和为某一值的路径](#二叉树中和为某一值的路径)
  * [二叉树的下一个节点](#二叉树的下一个节点)
  * [对称的二叉树](#对称的二叉树)
  * [把二叉树打印成多行](#把二叉树打印成多行)
  * [按之字形顺序打印二叉树](#按之字形顺序打印二叉树)
  * [序列化和反序列化二叉树](#序列化和反序列化二叉树)
  * [二叉搜索树的第k个结点](#二叉搜索树的第k个结点)
* [栈和队列](#栈和队列)
  * [用两个队列实现栈，用两个栈实现队列。](#用两个队列实现栈，用两个栈实现队列。)
  * [包含min函数的栈](#包含min函数的栈)
  * [栈的压入和弹出序列](#栈的压入和弹出序列)
* [排序和查找](#排序和查找)
  * [旋转数组的最小数字](#旋转数组的最小数字)
* [递归](#递归)
  * [斐波那契数列](#斐波那契数列)
  * [青蛙跳台阶](#青蛙跳台阶)
  * [变态跳台阶](#变态跳台阶)
  * [矩形覆盖](#矩形覆盖)
* [位运算](#位运算)
  * [二进制中1的个数](#二进制中1的个数)
  * [数组中只出现一次的数字](#数组中只出现一次的数字)
  * [不用加减乘除做加法](#不用加减乘除做加法)
* [回溯和DFS](#回溯和dfs)
  * [矩阵中的路径](#矩阵中的路径)
  * [机器人的运动范围](#机器人的运动范围)



点击关注[公众号](#公众号)及时获取笔主最新更新文章，并可免费领取Java工程师必备学习资源。

[TOC]

节选剑指offer比较经典和巧妙的一些题目，以便复习使用。一部分题目给出了完整代码，一部分题目比较简单直接给出思路。但是不保证我说的思路都是正确的，个人对算法也不是特别在行，只不过这本书的算法多看了几遍多做了几遍多了点心得体会。于是想总结一下。如果有错误也希望能指出，谢谢。

具体代码可以参考我的GitHub仓库：

https://github.com/h2pl/SwordToOffer
<!-- more -->


# 数论和数字规律

## 从1到n整数中1出现的次数

题目描述
求出1~13的整数中1出现的次数,并算出100~1300的整数中1出现的次数？为此他特别数了一下1~13中包含1的数字有1、10、11、12、13因此共出现6次,但是对于后面问题他就没辙了。ACMer希望你们帮帮他,并把问题更加普遍化,可以很快的求出任意非负整数区间中1出现的次数。

1暴力办法，把整数转为字符串，依次枚举相加。复杂度是O（N * k）k为数字长度。

2第二种办法看不懂，需要数学推导，太长不看

## 排数组排成最小的数

输入一个正整数数组，把数组里所有数字拼接起来排成一个数，打印能拼接出的所有数字中最小的一个。例如输入数组{3，32，321}，则打印出这三个数字能排成的最小数字为321323。

解析：本题的关键是，两个数如何排成最小的，答案是，如果把数字看成字符串a,b那么如果a+b>b+a，则a应该放在b后面。
例如 3和32 3 + 32 = 332,32 + 3 = 323,332>323,所以32要放在前面。

根据这个规律，构造一个比较器，使用排序方法即可。

## 丑数

题目描述
把只包含因子2、3和5的数称作丑数（Ugly Number）。例如6、8都是丑数，但14不是，因为它包含因子7。 习惯上我们把1当做是第一个丑数。求按从小到大的顺序的第N个丑数。

解析

1 暴力枚举每个丑数，找出第N个即可。

2 这个思路比较巧妙，由于丑数一定是由2,3,5三个因子构成的，所以我们每次构造出一个比前面丑数大但是比后面小的丑数，构造N次即可。

		    public class Solution {
		        public static int GetUglyNumber_Solution(int index) {
		                if (index == 0) return 0;
		                int []res = new int[index];
		                res[0] = 1;
		                int i2,i3,i5;
		                i2 = i3 = i5 = 0;
		                for (int i = 1;i < index;i ++) {
		                    res[i] = Math.min(res[i2] * 2, Math.min(res[i3] * 3, res[i5] * 5));
		                    if (res[i] == res[i2] * 2) i2 ++;
		                    if (res[i] == res[i3] * 3) i3 ++;
		                    if (res[i] == res[i5] * 5) i5 ++;
		                }
		                return res[index - 1];
		            }
		        }
		    }
		    i2,i3,i5分别代表目前有几个2,3,5的因子，每次选一个最小的丑数，然后开始找下一个。当然i2，i3，i5也要跟着变。
# 数组和矩阵


## 二维数组的查找

    /**
     * Created by 周杰伦 on 2018/2/25.
     * 题目描述
     在一个二维数组中，每一行都按照从左到右递增的顺序排序，
     每一列都按照从上到下递增的顺序排序。请完成一个函数，
     输入这样的一个二维数组和一个整数，判断数组中是否含有该整数。
     1 2 3
     2 3 4
     3 4 5
     */
     
     解析：比较经典的一题，解法也比较巧妙，由于数组从左向右和从上到下的都是递增的，所以找一个数可以先从最右开始找。
     假设最右值为a，待查数为x，那么如果x < a说明x在a的左边，往左找即可，如果x > a，说明x 在 a的下面一行，到下面一行继续按照该规则查找，就可以遍历所有数。
     
    算法的时间复杂度是O(M * N)
     
    public class 二维数组中的查找 {
        public static boolean Find(int target, int[][] array) {
    
            if(array[0][0] > target) {
                return false;
            }
    
            int row = 0;
            int col = 0;
            while (row < array.length && col >0) {
                if (target == array[row][col]) {
                    return true;
                }
                else if (target <array[row][col]) {
                    col --;
                }
                else if (target > array[row][col]) {
                    col ++;
                }
                else row++;
            }
            return false;
        }
    }
## 顺时针打印矩阵。

输入一个矩阵，按照从外向里以顺时针的顺序依次打印出每一个数字，例如，如果输入如下矩阵： 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 则依次打印出数字1,2,3,4,8,12,16,15,14,13,9,5,6,7,11,10.

这题还是有点麻烦的，因为要顺时针打印，所以实际上是由外向内打印，边界的处理和递归调用需要谨慎。

这题我自己没写出标准答案。参考一个答案吧。关键在于四个循环中的分界点设置。

    //主体循环部分才5行。其实是有规律可循的。将每一层的四个边角搞清楚就可以打印出来了
            
        import java.util.ArrayList;
        public class Solution {
            public ArrayList<Integer> printMatrix(int [][] array) {
                ArrayList<Integer> result = new ArrayList<Integer> ();
                if(array.length==0) return result;
                int n = array.length,m = array[0].length;
                if(m==0) return result;
                int layers = (Math.min(n,m)-1)/2+1;//这个是层数
                for(int i=0;i<layers;i++){
                    for(int k = i;k<m-i;k++) result.add(array[i][k]);//左至右
                    for(int j=i+1;j<n-i;j++) result.add(array[j][m-i-1]);//右上至右下
                    for(int k=m-i-2;(k>=i)&&(n-i-1!=i);k--) result.add(array[n-i-1][k]);//右至左
                    for(int j=n-i-2;(j>i)&&(m-i-1!=i);j--) result.add(array[j][i]);//左下至左上
                }
                return result;       
            }
        }

## 调整数组中数字的顺序，使正数在负数的前面

双指针即可以解决，变式有正负，奇偶等等。

## 数组中出现次数超过一半的数字

本题有很多种解法。

1 最笨的解法，统计每个数的出现次数，O（n2)

2 使用hashmap，空间换时间O（n)

3 由于出现超过一半的数字一定也是中位数，所以可以先排序，再找到第n/2位置上的节点。

4 使用快速排序的复杂度是O（nlogn)，基于快排的特性，每一轮的过程都会把一个数放到最终位置，所以我们可以判断一下这个数的位置是不是n/2，如果是的话，那么就直接返回即可。这样就优化了快排的步骤。

4.5事实上，上述办法的复杂度仍然是O（nlogn）

    快速排序的partition函数将一个数组分为左右两边，并且我们可以知道，如果flag值在k位置左边，那么往左找，如果在k位置右边，那么往左找。
    
    这里科普一下经典快排中的一个方法partition，剑指offer书中直接跳过了这部分，让我摸不着头脑。
    
    虽然快排用到了经典的分而治之的思想，但是快排实现的前提还是在于 partition 函数。正是有了 partition 的存在，才使得可以将整个大问题进行划分，进而分别进行处理。
    
    除了用来进行快速排序，partition 还可以用 O(N) 的平均时间复杂度从无序数组中寻找第K大的值。和快排一样，这里也用到了分而治之的思想。首先用 partition 将数组分为两部分，得到分界点下标 pos，然后分三种情况：
    
    pos == k-1，则找到第 K 大的值，arr[pos]；
    pos > k-1，则第 K 大的值在左边部分的数组。
    pos < k-1，则第 K 大的值在右边部分的数组。
    下面给出基于迭代的实现（用来寻找第 K 小的数）：


    int find_kth_number(vector<int> &arr, int k){
        int begin = 0, end = arr.size();
        assert(k>0 && k<=end);
        int target_num = 0;
        while (begin < end){
            int pos = partition(arr, begin, end);
            if(pos == k-1){
                target_num = arr[pos];
                break;
            }
            else if(pos > k-1){
                end = pos;
            }
            else{
                begin = pos + 1;
            }
        }
        return target_num;
    }

   

    该算法的时间复杂度是多少呢？考虑最坏情况下，每次 partition 将数组分为长度为 N-1 和 1 的两部分，然后在长的一边继续寻找第 K 大，此时时间复杂度为 O(N^2 )。不过如果在开始之前将数组进行随机打乱，那么可以尽量避免最坏情况的出现。而在最好情况下，每次将数组均分为长度相同的两半，运行时间 T(N) = N + T(N/2)，时间复杂度是 O(N)。

所以也就是说，本题用这个方法解的话，复杂度只需要O（n),因为第一次交换需要N/2,j接下来的交换的次数越来越少，最后加起来就是O（N）了。

5 由于数字出现次数超过长度的一半，也就是平均每两个数字就有一个该数字，但他们不一定连续，所以变量time保存一个数的出现次数，然后变量x代表目前选择的数字，遍历中，如果x与后一位不相等则time--，time=0时x改为后一位，time重新变为1。最终x指向的数字就是出现次数最多的。

举两个例子，比如1,2,3,4,5,6,6,6,6,6,6。明显符合。1,6,2,6,3,6,4,6,5,6,6 遍历到最后得到x=6，以此类推，可以满足要求。

## 找出前k小的数
 * 输入n个整数，找出其中最小的K个数。例如输入4,5,1,6,2,7,3,8这8个数字，则最小的4个数字是1,2,3,4,。
 */

解析：

1如果允许改变数组，那么则可以继承上一题的思想。，使用快速排序中的partition方法，只需要O（N）的复杂度

2使用堆排序

    解析：用前k个数构造一个大小为k的大顶堆，然后遍历余下数字，如果比堆顶大，则跳过，如果比堆顶小，则替换掉堆顶元素，然后执行一次堆排序（即根节点向下调整)。此时的堆顶元素已被替换，
    
    然后遍历完所有元素，堆中的元素就是最小的k个元素了。
    
    如果要求最大的k个元素，则构造小顶堆就可以了。
    
    构造堆的方法是，数组的第N/2号元素到0号元素依次向下调整，此时数组就构成了堆。
    
    实际上我们可以使用现成的集合类，红黑树是一棵搜索树，他是排序的，所以可以得到最大和最小值，那么我们每次和最小值比较，符合条件就进行替换即可。复杂度是O（nlogn)


    public ArrayList<Integer> GetLeastNumbers_Solution(int [] input, int k) {
    
            ArrayList<Integer>arrayList=new ArrayList<>();
            if(input==null || input.length==0 ||k==0 ||k>input.length)return arrayList;
    
            TreeSet<Integer> treeSet=new TreeSet<>();


​    
            for(int i=0;i<input.length;i++){
                if(treeSet.size()<k){
                    treeSet.add(input[i]);
                }
    
                else {
    
                    if(input[i]<treeSet.last()){
                        treeSet.pollLast();
                        treeSet.add(input[i]);
                    }
                }
            }
    
            for(Integer x:treeSet){
            arrayList.add(x);
    
            }
            return arrayList;
    
        }

## 连续子数组的最大和

**
 * Created by 周杰伦 on 2017/3/23.
 * 题目描述
 HZ偶尔会拿些专业问题来忽悠那些非计算机专业的同学。
 今天测试组开完会后,他又发话了:在古老的一维模式识别中,常常需要计算连续子向量的最大和,当向量全为正数的时候,问题很好解决。
 但是,如果向量中包含负数,是否应该包含某个负数,并期望旁边的正数会弥补它呢？
 例如:{6,-3,-2,7,-15,1,2,2},连续子向量的最大和为8(从第0个开始,到第3个为止)。
 你会不会被他忽悠住？(子向量的长度至少是1)


解析：笨办法需要O（n2）的复杂度。

1 但是实际上只需要
一次遍历即可解决。通过sum保存当前和，然后如果当前和为正，那么继续往后加，如果当前和为负，则直接丢弃，令当前和等于一个新值。并且每一步都要比较当前和与最大值。

     */
    
    public class 连续数字序列的最大和 {
        public int FindGreatestSumOfSubArray(int[] array) {
           if(array==null || array.length==0)return 0;
            int sum=0;int max=array[0];
    
            for(int i=0;i<array.length;i++){
               //如果当前和<0，那就不加，直接赋新值
                if(sum<=0){
                   sum=array[i];
               }//如果当前和大于零，则继续加。
               else {
                   sum+=array[i];
               }
               if(max<sum){
                max=sum;
               }
    
            }
            return max;
        }

2 本题也可以使用DP解法      

DP数组代表以i为结尾元素的连续最大和

DP[i] = arr[i] (DP[i-1] < 0) 
      = DP[i] + arr[i] (DP[i -1] > 0)
      

## 逆序对

/**
 * Created by 周杰伦 on 2017/3/23.
 * 题目描述
 在数组中的两个数字，如果前面一个数字大于后面的数字，则这两个数字组成一个逆序对。
 输入一个数组,求出这个数组中的逆序对的总数P。并将P对1000000007取模的结果输出。
 即输出P%1000000007
 */

    解析：本题采用归并排序的框架，只是在归并的时候做出逆序对查找，具体参见下面代码。
    核心点是，在归并两个有序数组时，如果a数组的元素a1比b数组的元素b1大时，说明有mid - i + 1个数都比b1大。i为a1元素的位置。
    这样子我们就可以统计逆序对的个数了。经典巧妙。！

    public class 逆序对 {
        public double Pairs = 0;
        public int InversePairs(int [] array) {
            if (array.length==0 ||array==null)
                return 0;
            mergesort(array,0,array.length-1);
            Pairs = Pairs + 1000000007;
            return (int) (Pairs % 1000000007);
        }
        public void merge(int []array,int left,int mid,int right){
            //有一点很重要的是，归并分成两部分，其中一段是left到mid，第二段是mid+1到right。
            //不能从0到mid-1，然后mid到right。因为这样左右不均分，会出错。千万注意。
            //mid=(left+right)/2
            if (array.length==0 ||array==null ||left>=right)
                return ;
            int p=left,q=mid+1,k=0;
    
            int []temp=new int[right-left+1];
        
            while (p<=mid && q<=right){
                if(array[p]>array[q]){
                    temp[k++]=array[q++];
                    //当前半数组中有一个数p比后半个数组中的一个数q大时，由于两个数组
                    //已经分别有序，所以说明p到中间数之间的所有数都比q大。
                    Pairs+=mid-p+1;
                }
                else temp[k++]=array[p++];
            }
        
            while (p<=mid){
                temp[k++]=array[p++];}
            while (q<=right){
                temp[k++]=array[q++];}
    
    
    
            for (int m = 0; m < temp.length; m++)
                array[left + m] = temp[m];
        
        }
        
        public void mergesort(int []arr,int left,int right){
            if (arr.length==0 ||arr==null)
                return ;
            int mid=(right+left)/2;
            if(left<right) {
                mergesort(arr, left, mid);
                mergesort(arr, mid + 1, right);
                merge(arr, left,mid, right);
            }
        }
## 数字在排序数组中出现的次数

1 顺序扫描

鲁迅说过：看到排序数组要想到二分法！

2 通过二分查找找到数字k第一次出现的位置，即先比较k和中间值，再依次二分，如果中间值等于k并且中间值左边！=k，则是第一个k。
反之可以找到最后一次出现k的位置。然后相减即可。复杂度是logn

## 和为s的两个整数，和为s的连续正数序列

1 和为s的两个整数，双指针遍历即可

2 和为s的连续正数序列。维护一个范围，start到end表示目前的数字序列。大于S则start++,小于S则start--

## n个色子的点数

求n个色子点数之和等于s的概率

1 递归实现

2 循环实现，所有可能值存成一个数组，大小为6n，然后把每个出现数字次数存入数组，遍历一遍即可得到概率。

## 扑克牌的顺子

题目描述
LL今天心情特别好,因为他去买了一副扑克牌,发现里面居然有2个大王,2个小王(一副牌原本是54张^_^)...他随机从中抽出了5张牌,想测测自己的手气,看看能不能抽到顺子,如果抽到的话,他决定去买体育彩票,嘿嘿！！“红心A,黑桃3,小王,大王,方片5”,“Oh My God!”不是顺子.....LL不高兴了,他想了想,决定大\小 王可以看成任何数字,并且A看作1,J为11,Q为12,K为13。上面的5张牌就可以变成“1,2,3,4,5”(大小王分别看作2和4),“So Lucky!”。LL决定去买体育彩票啦。 现在,要求你使用这幅牌模拟上面的过程,然后告诉我们LL的运气如何。为了方便起见,你可以认为大小王是0。

把扑克牌存到数组中，并且A看作1,J为11,Q为12,K为13，然后进行排序，如果有不连续的数字，不存在顺子，如果都连续，则是顺子

     Arrays.sort(arr);
            for (int i = 1 ;i < arr.length;i ++) {
                if (arr[i] == arr[i - 1]) {
                    return false;
                }
                if (arr[i] - arr[i - 1] == 1) {
                    continue;
                }else if (arr[i] - arr[i - 1]  - 1 <= cnt) {
                    cnt -= arr[i] - arr[i - 1] - 1;
                }else {
                    return false;
                }
            }
            return true;
        }

## 数组中重复的数字

在一个长度为n的数组里的所有数字都在0到n-1的范围内。 数组中某些数字是重复的，但不知道有几个数字是重复的。也不知道每个数字重复几次。请找出数组中任意一个重复的数字。 例如，如果输入长度为7的数组{2,3,1,0,2,5,3}，那么对应的输出是第一个重复的数字2。

## 数组中重复的数字

在一个长度为n的数组里的所有数字都在0到n-1的范围内。 数组中某些数字是重复的，但不知道有几个数字是重复的。也不知道每个数字重复几次。请找出数组中任意一个重复的数字。 例如，如果输入长度为7的数组{2,3,1,0,2,5,3}，那么对应的输出是第一个重复的数字2。

解析：一般使用hashmap即可达到O（n)

但剑指offer的解法可以只用O(1)的空间做到。

实际上当每个数字a和他们所在的位置n相同时，每个数字只出现一次，但如果n + 1的位置上的数也是a，那么a就是第一个重复出现的数字。

根据这个思路。我们在循环中当第i个数a与arr[a]相等时，不变，如果不相等，则两者互换，然后开始下一轮遍历。接下来继续交换，如果出现相等的情况，则就是第一个重复出现的数。

举例 2 3 1 0 2 5 3
1 arr[0] = 2 != 0,所以arr[0]与arr[2]做交换，得1 3 2 0 2 5 3
2 arr[0] = 1 != 0，所以arr[0]和arr[1]交换，的3 1 2 0 2 5 3
3 arr[0] = 3 != 0,所以arr[0]和arr[3]交换，得0 1 2 3 2 5 3
4 arr[0]到arr[3]都符合要求,arr[4] = 2 != 4，所以arr[4]和arr[2]交换，发现两者相等，所以他就是第一个重复的数。


    public boolean duplicate(int array[],int length,int [] duplication) {
        if ( array==null ) return false;
    
        // key step
        for( int i=0; i<length; i++ ){
            while( i!=array[i] ){
                if ( array[i] == array[array[i]] ) {
                    duplication[0] = array[i];
                    return true;
                }
    
                int temp = array[i];
                array[i] = array[array[i]];
                array[array[i]] = temp;
            }
        }
    
        return false;
    }

从上述例子可以看到，一个数最多被交换两次。所以复杂度为O（N)

## 构建乘积数组

题目描述
给定一个数组A[0,1,...,n-1],请构建一个数组B[0,1,...,n-1],其中B中的元素B[i]=A[0]*A[1]*...*A[i-1]*A[i+1]*...*A[n-1]。不能使用除法。

不能用除法，那么就两个循环，一个从0乘到i - 1，一个从i + 1乘到n-1

## 数据流的中位数

题目描述
如何得到一个数据流中的中位数？如果从数据流中读出奇数个数值，那么中位数就是所有数值排序之后位于中间的数值。如果从数据流中读出偶数个数值，那么中位数就是所有数值排序之后中间两个数的平均值。

解析，与字符流第一个不重复的字符类似，每次添加数字都要输出一次结果。

    public class Solution {
    
        static ArrayList<Integer> list = new ArrayList<>();
        public static void Insert(Integer num) {
            list.add(num);
            Collections.sort(list);
        }
    
        public static Double GetMedian() {
            if (list.size() % 2 == 0) {
                int l = list.get(list.size()/2);
                int r = list.get(list.size()/2 - 1);
                return (l + r)/2.0;
            }
            else {
                return list.get(list.size()/2)/1.0;
            }
        }


​    
    }

## 滑动窗口中的最大值

给定一个数组和滑动窗口的大小，找出所有滑动窗口里数值的最大值。例如，如果输入数组{2,3,4,2,6,2,5,1}及滑动窗口的大小3，那么一共存在6个滑动窗口，他们的最大值分别为{4,4,6,6,6,5}； 针对数组{2,3,4,2,6,2,5,1}的滑动窗口有以下6个： {[2,3,4],2,6,2,5,1}， {2,[3,4,2],6,2,5,1}， {2,3,[4,2,6],2,5,1}， {2,3,4,[2,6,2],5,1}， {2,3,4,2,[6,2,5],1}， {2,3,4,2,6,[2,5,1]}。

解析：
1 保持窗口为3进行右移，每次计算出一个最大值即可。

2 使用两个栈实现一个队列，复杂度O（N）,使用两个栈实现最大值栈，复杂度O（1）。两者结合可以完成本题。但是太麻烦了。

3 使用双端队列解决该问题。

    import java.util.*;
    /**
    用一个双端队列，队列第一个位置（队头）保存当前窗口的最大值，当窗口滑动一次
    1.判断当前最大值是否过期（如果最大值所在的下标已经不在窗口范围内，则过期）
    2.对于一个新加入的值，首先一定要先放入队列，即使他比队头元素小，因为队头元素可能过期。
    3.新增加的值从队尾开始比较，把所有比他小的值丢掉(因为队列只存最大值，所以之前比他小的可以丢掉)
    
    */
    public class Solution {
       public ArrayList<Integer> maxInWindows(int [] num, int size)
        {
            ArrayList<Integer> res = new ArrayList<>();
            if(size == 0) return res;
            int begin; 
            ArrayDeque<Integer> q = new ArrayDeque<>();
            for(int i = 0; i < num.length; i++){
                begin = i - size + 1;
                if(q.isEmpty())
                    q.add(i);
                else if(begin > q.peekFirst())
                    q.pollFirst();
             
                while((!q.isEmpty()) && num[q.peekLast()] <= num[i])
                    q.pollLast();
                q.add(i);  
                if(begin >= 0)
                    res.add(num[q.peekFirst()]);
            }
            return res;
        }
    }

# 字符串

## 字符串的排列

输入一个字符串,按字典序打印出该字符串中字符的所有排列。例如输入字符串abc,则打印出由字符a,b,c所能排列出来的所有字符串abc,acb,bac,bca,cab和cba。

    解析：这是一个全排列问题，也就是N个不同的数排成所有不同的序列，只不过把数换成了字符串。
    全排列的过程就是，第一个元素与后续的某个元素交换，然后第二个元素也这么做，直到最后一个元素为之，过程是一个递归的过程，也是一个dfs的过程。
    
    注意元素也要和自己做一次交换，要不然会漏掉自己作为头部的情况。
    
    然后再进行一次字典序的排序即可。
    
    public static ArrayList<String> Permutation(String str) {
            char []arr = str.toCharArray();
            List<char []> list = new ArrayList<>();
            all(arr, 0, arr.length - 1, list);
            Collections.sort(list, (o1, o2) -> String.valueOf(o1).compareTo(String.valueOf(o2)));
            ArrayList<String> res = new ArrayList<>();
            for (char[] c : list) {
                if (!res.contains(String.valueOf(c)))
                res.add(String.valueOf(c));
            }
            return res;
        }
    
        //注意要换完为之，因为每换一次可以去掉头部一个数字，这样可以避免重复
        public static void all(char []arr, int cur, int end, List<char[]> list) {
            if (cur == end) {
    //            System.out.println(Arrays.toString(arr));
                list.add(Arrays.copyOf(arr, arr.length));
            }
            for (int i = cur;i <= end;i ++) {
                //这里的交换包括跟自己换，所以只有一轮换完才能确定一个结果
                swap(arr, cur, i);
                all(arr, cur + 1, end, list);
                swap(arr, cur, i);
            }
        }
        public static void swap(char []arr, int i, int j) {
            if (i > arr.length || j > arr.length || i >= j)return;
            char temp = arr[i];
            arr[i] = arr[j];
            arr[j] = temp;
        }

## 替换空格

    /**
     * Created by 周杰伦 on 2018/2/25.
     * 请实现一个函数，将一个字符串中的空格替换成“%20”。例如，当字符串为We Are Happy.则经过替换之后的字符串为We%20Are%20Happy。
     */
     
     解析：如果单纯地按顺序替换空格，每次替换完还要将数组扩容，再右移，这部操作的时间复杂度就是O（2*N）=O（N)，所以总的复杂度是O（n^2)，所以应该采取先扩容的办法，统计出空格数，然后扩容，接下来按顺序添加字符，遇到空格直接改成添加%20即可，这样避免了右移操作和多次扩容，复杂度是O（N)


​     
    public class 替换空格 {
        public static String replaceSpace(StringBuffer str) {
            int newlen = 0;
            for(int i = 0; i < str.length(); i++) {
                if(str.charAt(i) == ' ') {
                    newlen = newlen + 3;
                }
                else {
                    newlen ++;
                }
            }
            char []newstr = new char[newlen];
            int j = 0;
            for(int i = 0 ; i < str.length(); i++) {
                if (str.charAt(i) == ' ') {
                    newstr[j++] = '%';
                    newstr[j++] = '2';
                    newstr[j++] = '0';
                }else {
                    newstr[j++] = str.charAt(i);
                }
            }
            return String.valueOf(newstr);
        }


## 第一次只出现一次的字符

哈希表可解

## 翻转单词顺序和左旋转字符串

1
题目描述
牛客最近来了一个新员工Fish，每天早晨总是会拿着一本英文杂志，写些句子在本子上。同事Cat对Fish写的内容颇感兴趣，有一天他向Fish借来翻看，但却读不懂它的意思。例如，“student. a am I”。后来才意识到，这家伙原来把句子单词的顺序翻转了，正确的句子应该是“I am a student.”。Cat对一一的翻转这些单词顺序可不在行，你能帮助他么？

    这个解法很经典，先把每个单词逆序，再把整个字符串逆序，结果就是把每个单词都进行了翻转。

2
汇编语言中有一种移位指令叫做循环左移（ROL），现在有个简单的任务，就是用字符串模拟这个指令的运算结果。对于一个给定的字符序列S，请你把其循环左移K位后的序列输出。例如，字符序列S=”abcXYZdef”,要求输出循环左移3位后的结果，即“XYZdefabc”。是不是很简单？OK，搞定它！

    字符串循环左移N位的处理方法也很经典，先把前N位逆序，再把剩余字符串逆序，最后整体逆序。
    
    abcXYZdef -> cbafedZYX -> XYZdefabc

## 把字符串转换为整数

题目描述
将一个字符串转换成一个整数，要求不能使用字符串转换整数的库函数。 数值为0或者字符串不是一个合法的数值则返回0

解析：首先需要判断正负号，然后判断每一位是否是数字，然后判断是否溢出，判断溢出可以通过加完第n位的和与未加第n位的和进行比较。最后可以得出结果。所以需要3-4步判断。


## 表示数值的字符串

请实现一个函数用来判断字符串是否表示数值（包括整数和小数）。例如，字符串"+100","5e2","-123","3.1416"和"-1E-16"都表示数值。 但是"12e","1a3.14","1.2.3","+-5"和"12e+4.3"都不是。

    不得不说这种题型太恶心了，就是需要一直判断边界条件
    
    参考一个答案。比较完整
    
        bool isNumeric(char* str) {
            // 标记符号、小数点、e是否出现过
            bool sign = false, decimal = false, hasE = false;
            for (int i = 0; i < strlen(str); i++) {
                if (str[i] == 'e' || str[i] == 'E') {
                    if (i == strlen(str)-1) return false; // e后面一定要接数字
                    if (hasE) return false;  // 不能同时存在两个e
                    hasE = true;
                } else if (str[i] == '+' || str[i] == '-') {
                    // 第二次出现+-符号，则必须紧接在e之后
                    if (sign && str[i-1] != 'e' && str[i-1] != 'E') return false;
                    // 第一次出现+-符号，且不是在字符串开头，则也必须紧接在e之后
                    if (!sign && i > 0 && str[i-1] != 'e' && str[i-1] != 'E') return false;
                    sign = true;
                } else if (str[i] == '.') {
                  // e后面不能接小数点，小数点不能出现两次
                    if (hasE || decimal) return false;
                    decimal = true;
                } else if (str[i] < '0' || str[i] > '9') // 不合法字符
                    return false;
            }
            return true;
        }

## 字符流中第一个不重复的字符

题目描述
请实现一个函数用来找出字符流中第一个只出现一次的字符。例如，当从字符流中只读出前两个字符"go"时，第一个只出现一次的字符是"g"。当从该字符流中读出前六个字符“google"时，第一个只出现一次的字符是"l"。

本题主要要注意的是流。也就是说每次输入一个字符就要做一次判断。比如输入aaaabbbcd，输出就是a###b##cd

    StringBuilder sb = new StringBuilder();
        int []map = new int[256];
        public  void Insert(char ch)
        {
            sb.append(ch);
            if (map[ch] == 0) {
                map[ch] = 1;
            }else {
                map[ch] ++;
            }
            System.out.println(FirstAppearingOnce());
            
        }
        //return the first appearence once char in current stringstream
        public char FirstAppearingOnce()
        {
            for (int i = 0;i < sb.length();i ++) {
                if (map[sb.charAt(i)] == 1) {
                    return sb.charAt(i);
                }
            }
            return '#';
        }

# 链表

## 从尾到头打印链表

考查递归，递归可以使输出的顺序倒置

    public static void printReverse(Node node) {
            if (node.next != null) {
                printReverse(node.next);
            }
            System.out.print(node.val + " ");
    }


## 链表倒数第k个节点

使用两个指针，一个先走k步。然后一起走即可。

## 反转链表

老生常谈，但是容易写错。

    public ListNode ReverseList(ListNode head) {
            if(head==null || head.next==null)return head;
            ListNode pre,next;
            pre=null;
            next=null;
            while(head!=null){
                //保存下一个结点
                next=head.next;
                //连接下一个结点
                head.next=pre;
                pre=head;
                head=next;
            }
            return pre;
        }
    }

## 合并两个排序链表

与归并排序的合并类似

## 复杂链表的复制

题目描述
输入一个复杂链表（每个节点中有节点值，以及两个指针，一个指向下一个节点，另一个特殊指针指向任意一个节点），返回结果为复制后复杂链表的head。（注意，输出结果中请不要返回参数中的节点引用，否则判题程序会直接返回空）

这题比较恶心。

解析：

1 直接复制链表，然后再去复制特殊指针，复杂度是O(n2)

2 使用hash表保存特殊指针的映射关系，第二步简化操作，复杂度是O（n）

3 复制每个节点并且连成一个大链表A-A'-B-B'，然后从头到尾判断特殊指针，如果有特殊指针，则让后续节点的特殊指针指向原节点特殊指针指向的节点的后置节点，晕了吧，其实就是原来是A指向B，现在是A’指向B‘。

最后我们根据奇偶序号把链表拆开，复杂度是O（N)且不用额外空间。

## 两个链表的第一个公共节点

1 逆置链表，反向找第一个不同节点，前一个就是公共节点

2 求长度并相减得n，短的链表先走n步，然后一起走即可。

## 孩子们的游戏(圆圈中最后剩下的数)

这是一个约瑟夫环问题。

1 使用循环链表求解，每次走n步摘取一个节点，然后继续，直到最后一个节点就是剩下的数，空间复杂度为O（n)

2 使用数组来做
public static int LastRemaining_Solution(int n, int m) {
        int []arr = new int[n];
        for (int i = 0;i < n;i ++) {
            arr[i] = i;
        }

        int cnt = 0;
        int sum = 0;
        for (int i = 0;i < n;i = (i + 1) % n) {
            if (arr[i] == -1) {
                continue;
            }
            cnt ++;
            if (cnt == m) {
                arr[i] = -1;
                cnt = 0;
                sum ++;
            }
            if (sum == n) {
                return i;
            }
        }
        return n - 1;
    }

3 使用余数法求解


    int LastRemaining_Solution(int n, int m) {
            if (m == 0 || n == 0) {
                return -1;
            }
            ArrayList<Integer> data = new ArrayList<Integer>();
            for (int i = 0; i < n; i++) {
                data.add(i);
            }
            int index = -1;
            while (data.size() > 1) {
    //          System.out.println(data);
                index = (index + m) % data.size();
    //          System.out.println(data.get(index));
                data.remove(index);
                index--;
            }
            return data.get(0);
        }

## 链表的环的入口结点

一个链表中包含环，请找出该链表的环的入口结点。

解析：

1 指定两个指针，一个一次走两步，一个一次走一步，然后当两个节点相遇时，这个节点必定在环中。既然这个节点在环中，那么让这个节点走一圈直到与自己相等为之，可以得到环的长度n。

2 得到了环的长度以后，根据数学推导的结果，我们可以指定两个指针，一个先走n步，然后两者同时走，这样的话，当慢节点到达入口节点时，快节点也转了一圈刚好又到达入口节点，所以也就是他们相等的时候就是入口节点了。

## 删除链表中重复的节点

题目描述
在一个排序的链表中，存在重复的结点，请删除该链表中重复的结点，重复的结点不保留，返回链表头指针。 例如，链表1->2->3->3->4->4->5 处理后为 1->2->5

保留头结点，然后找到下一个不重复的节点，与他相连，重复的节点直接跳过即可。

#二叉树

## 二叉搜索树转换为双向链表

输入一棵二叉搜索树，将该二叉搜索树转换成一个排序的双向链表。要求不能创建任何新的结点，只能调整树中结点指针的指向。

二叉搜索树要转换成有序的双向链表，实际上就是使用中序遍历把节点连入链表中，并且题目要求在原来节点上进行操作，也就是使用左指针和右指针表示链表的前置节点和后置节点。

使用栈实现中序遍历的非递归算法，便可以找出节点的先后关系，依次连接即可。

    public TreeNode Convert(TreeNode root) {
            if(root==null)
                return null;
            Stack<TreeNode> stack = new Stack<TreeNode>();
            TreeNode p = root;
            TreeNode pre = null;// 用于保存中序遍历序列的上一节点
            boolean isFirst = true;
            while(p!=null||!stack.isEmpty()){
                while(p!=null){
                    stack.push(p);
                    p = p.left;
                }
                p = stack.pop();
                if(isFirst){
                    root = p;// 将中序遍历序列中的第一个节点记为root
                    pre = root;
                    isFirst = false;
                }else{
                    pre.right = p;
                    p.left = pre;
                    pre = p;
                }      
                p = p.right;
            }
            return root;
        }
    }

## 重建二叉树

         * 题目描述
     输入某二叉树的前序遍历和中序遍历的结果，请重建出该二叉树。假设输入的前序遍历和中序遍历的结果中都不含重复的数字。例如输入前序遍历序列{1,2,4,7,3,5,6,8}和中序遍历序列{4,7,2,1,5,3,8,6}，则重建二叉树并返回。
     */
     
     解析：首先，头结点一定是先序遍历的首位，并且该节点把中序分为左右子树，根据这个规则，左子树由左边数组来完成，右子树由右边数组来完成，根节点由中间节点来构建，于是便有了如下的递归代码。该题的难点就在于边界的判断。
     
    public TreeNode reConstructBinaryTree(int [] pre, int [] in) {
        if(pre.length == 0||in.length == 0){
            return null;
        }
        TreeNode node = new TreeNode(pre[0]);
        for(int i = 0; i < in.length; i++){
            if(pre[0] == in[i]){
                node.left = reConstructBinaryTree(Arrays.copyOfRange(pre, 1, i+1), Arrays.copyOfRange(in, 0, i));//为什么不是i和i-1呢，因为要避免出错，中序找的元素需要再用一次。
                node.right = reConstructBinaryTree(Arrays.copyOfRange(pre, i+1, pre.length), Arrays.copyOfRange(in, i+1,in.length));
            }
        }
        return node;
    }

## 树的子结构

    /**
     * Created by 周杰伦 on 2018/3/27.
     * 输入两棵二叉树A，B，判断B是不是A的子结构。（ps：我们约定空树不是任意一个树的子结构）
     */
    
        解析：本题还是有点难度的，子结构要求节点完全相同，所以先判断节点是否相同，然后使用先序遍历进行递判断，判断的依据是如果子树为空，则说明节点都找到了，如果原树节点为空，说明找不到对应节点,接着递归地判断该节点的左右子树是否符合要求.
    
        public class 树的子结构 {
    
            public boolean HasSubtree(TreeNode root1, TreeNode root2) {
                boolean res = false;
                if (root1 != null && root2 != null) {
                    if (root1.val == root2.val) {
                        res = aHasb(root1, root2);
                    }
                    if (res == false) {
                        res = HasSubtree(root1.left,root2);
                    }
                    if (res == false) {
                        res = HasSubtree(root1.right,root2);
                    }
                    return res;
                }
                else return false;
            }
            public boolean aHasb(TreeNode t1, TreeNode t2){
                if (t2 == null) return true;
                if (t1 == null) return false;
                if (t1.val != t2.val) return false;
        
                return aHasb(t1.left,t2.left) && aHasb(t1.right,t2.right);
            }
        }


​    
## 镜像二叉树

    /**
     * Created by 周杰伦 on 2017/3/19.操作给定的二叉树，将其变换为源二叉树的镜像。
     输入描述:
     二叉树的镜像定义：源二叉树
      8
     /  \
     6   10
     / \  / \
     5  7 9 11
     镜像二叉树
       8
     /  \
     10   6
     / \  / \
     11 9 7  5
     */


    解析：其实镜像二叉树就是交换所有节点的左右子树，所以使用遍历并且进行交换即可。
    
    /**
     public class TreeNode {
     int val = 0;
     TreeNode left = null;
     TreeNode right = null;
    
     public TreeNode(int val) {
     this.val = val;
    
     }
    
     }
     */
    public class 镜像二叉树 {
        public void Mirror(TreeNode root) {
            if(root == null)return;
            if(root.left!=null || root.right!=null)
            {
                TreeNode temp=root.left;
                root.left=root.right;
                root.right=temp;
            }
            Mirror(root.left);
            Mirror(root.right);
    
        }


​    
## 树的层次遍历

也就是从上到下打印节点，使用队列即可完成。

## 二叉树的深度

经典遍历。

## 判断是否平衡二叉树

判断左右子树的高度差是否 <= 1即可。

## 二叉搜索树的后序遍历

题目描述
输入一个整数数组，判断该数组是不是某二叉搜索树的后序遍历的结果。如果是则输出Yes,否则输出No。假设输入的数组的任意两个数字都互不相同。

解析：这题其实也非常巧妙。二叉搜索树的特点就是他的左子树都比根节点小，右子树都比跟节点大。而后序遍历的根节点在最后，所以后续遍历的第1到N-1个节点应该是左右子树的节点（不一定左右子树都存在）。

后续遍历的序列是先左子树，再右子树，最后根节点，那么就要求，左半部分比根节点小，右半部分比根节点大，当然，左右部分不一定都存在。

所以，找出根节点后，首先找出左半部分，要求小于根节点，然后找出右半部分，要求大于根节点，如果符合，则递归地判断左右子树到的根节点（本步骤已经将左右部分划分，割据中间节点进行递归），如果不符合，直接返回false。

同理也可以判断前序遍历和中序遍历。

    public class 二叉搜索树的后序遍历序列 {
        public static void main(String[] args) {
            int []a = {7,4,6,5};
            System.out.println(VerifySquenceOfBST(a));
        }
        public static boolean VerifySquenceOfBST(int [] sequence) {
            if (sequence == null || sequence.length == 0) {
                return false;
            }
            return isBST(sequence, 0, sequence.length - 1);
        }
        public static boolean isBST(int []arr, int start, int end) {
            if (start >= end) return true;
            int root = arr[end];
            int mid = start;
            for (mid  = start;mid < end && arr[mid] < root;mid ++) {
    
            }
            for (int i = mid;i < end; i ++) {
                if (arr[i] < root)return false;
            }
            return isBST(arr, start, mid - 1) && isBST(arr, mid, end - 1);
        }
    }

## 二叉树中和为某一值的路径

/**
 * Created by 周杰伦 on 2018/3/29.
 * 题目描述
 输入一颗二叉树和一个整数，打印出二叉树中结点值的和为输入整数的所有路径。路径定义为从树的根结点开始往下一直到叶结点所经过的结点形成一条路径。
 */

    解析：由于要求从根节点到达叶子节点，并且要打印出所有路径，所以实际上用到了回溯的思想。
    
    通过target跟踪当前和，进行先序遍历，当和满足要求时，加入集合，由于有多种结果，所以需要回溯，将访问过的节点弹出访问序列，才能继续访问下一个节点。
    
    终止条件是和满足要求，并且节点是叶节点，或者已经访问到空节点也会返回。
    

    public class 二叉树中和为某一值的路径 {
        private ArrayList<ArrayList<Integer>> listAll = new ArrayList<ArrayList<Integer>>();
        private ArrayList<Integer> list = new ArrayList<Integer>();
        public ArrayList<ArrayList<Integer>> FindPath(TreeNode root,int target) {
            if(root == null) return listAll;
            list.add(root.val);
            target -= root.val;
            if(target == 0 && root.left == null && root.right == null)
                listAll.add(new ArrayList<Integer>(list));
            FindPath(root.left, target);
            FindPath(root.right, target);
            list.remove(list.size()-1);
            return listAll;
        }
    
        static int count = 0;
        static Stack<Integer> path = new Stack<>();
        static Stack<Integer> stack = new Stack<>();
        static ArrayList<ArrayList<Integer>> lists = new ArrayList<>();
    }

## 二叉树的下一个节点

给定一个二叉树和其中的一个结点，请找出中序遍历顺序的下一个结点并且返回。注意，树中的结点不仅包含左右子结点，同时包含指向父结点的指针。

    解析：给出一个比较好懂的解法，中序遍历的结果存在集合中，找到根节点，进行中序遍历，然后找到该节点，下一个节点就是集合后一位
    
    public TreeLinkNode GetNext(TreeLinkNode TreeLinkNode)
        {
            return findNextNode(TreeLinkNode);
        }
        public TreeLinkNode findNextNode(TreeLinkNode anynode) {
            if (anynode == null) return null;
            TreeLinkNode p = anynode;
            while (p.next != null) {
                p = p.next;
            }
            ArrayList<TreeLinkNode> list = inOrderSeq(p);
            for (int i = 0;i < list.size();i ++) {
                if (list.get(i) == anynode) {
                    if (i + 1 < list.size()) {
                        return list.get(i + 1);
                    }
                    else return null;
                }
            }
            return null;
    
        }
        static ArrayList<TreeLinkNode> list = new ArrayList<>();
        public static ArrayList<TreeLinkNode> inOrderSeq(TreeLinkNode TreeLinkNode) {
            if (TreeLinkNode == null) return null;
            inOrderSeq(TreeLinkNode.left);
            list.add(TreeLinkNode);
            inOrderSeq(TreeLinkNode.right);
            return list;
        }

## 对称的二叉树

请实现一个函数，用来判断一颗二叉树是不是对称的。注意，如果一个二叉树同此二叉树的镜像是同样的，定义其为对称的。

解析，之前有一题是二叉树的镜像，递归交换左右子树即可求出镜像，然后递归比较两个树的每一个节点，则可以判断是否对称。

    boolean isSymmetrical(TreeNode pRoot)
        {
            TreeNode temp = copyTree(pRoot);
            Mirror(pRoot);
            return isSameTree(temp, pRoot);
        }


​    
        void Mirror(TreeNode root) {
            if(root == null)return;
            Mirror(root.left);
            Mirror(root.right);
            if(root.left!=null || root.right!=null)
            {
                TreeNode temp=root.left;
                root.left=root.right;
                root.right=temp;
            }


​    
        }
        boolean isSameTree(TreeNode t1,TreeNode t2){
            if(t1==null && t2==null)return true;
            else if(t1!=null && t2!=null && t1.val==t2.val) {
                boolean left = isSameTree(t1.left, t2.left);
                boolean right = isSameTree(t1.right, t2.right);
                return left && right;
            }
            else return false;
        }
    
        TreeNode copyTree (TreeNode root) {
            if (root == null) return null;
            TreeNode t = new TreeNode(root.val);
            t.left = copyTree(root.left);
            t.right = copyTree(root.right);
            return t;
        }
## 把二叉树打印成多行

题目描述
从上到下按层打印二叉树，同一层结点从左至右输出。每一层输出一行。

解析：1 首先要知道到本题的基础思想，层次遍历。

2 然后是进阶的思想，按行打印二叉树并输出行号，方法是，一个节点last指向当前行的最后一个节点，一个节点nlast指向下一行最后一个节点。使用t表示现在遍历的节点，当t = last时，表示本行结束。此时last = nlast，开始下一行遍历。

同时，当t的左右子树不为空时，令nlast = t的左子树和右子树。每当last 赋值为nlast时，行号加一即可。

## 按之字形顺序打印二叉树

请实现一个函数按照之字形打印二叉树，即第一行按照从左到右的顺序打印，第二层按照从右至左的顺序打印，第三行按照从左到右的顺序打印，其他行以此类推。

解析：1 首先要知道到本题的基础思想，层次遍历。

2 然后是进阶的思想，按行打印二叉树并输出行号，方法是，一个节点last指向当前行的最后一个节点，一个节点nlast指向下一行最后一个节点。使用t表示现在遍历的节点，当t = last时，表示本行结束。此时last = nlast，开始下一行遍历。

同时，当t的左右子树不为空时，令nlast = t的左子树和右子树。每当last 赋值为nlast时，行号加一即可。

3 基于第2步的思想，现在要z字型打印，只需把偶数行逆序即可。所以把每一行的数存起来，然后偶数行逆置即可。

    ArrayList<ArrayList<Integer> > Print(TreeNode pRoot) {
            LinkedList<TreeNode> queue = new LinkedList<>();
            TreeNode root = pRoot;
            if(root == null) {
                return new ArrayList<>();
            }
            TreeNode last = root;
            TreeNode nlast = root;
            queue.offer(root);
            ArrayList<Integer> list = new ArrayList<>();
            list.add(root.val);
            ArrayList<Integer> one = new ArrayList<>();
            one.addAll(list);
            ArrayList<ArrayList<Integer>> lists = new ArrayList<>();
            lists.add(one);
            list.clear();
    
            int row = 1;
            while (!queue.isEmpty()){
    
                TreeNode t = queue.poll();
    
                if(t.left != null) {
                    queue.offer(t.left);
                    list.add(t.left.val);
                    nlast = t.left;
                }
                if(t.right != null) {
                    queue.offer(t.right);
                    list.add(t.right.val);
                    nlast = t.right;
                }
                if(t == last) {
                    if(!queue.isEmpty()) {
                        last = nlast;
                        row ++;
                        ArrayList<Integer> temp = new ArrayList<>();
                        temp.addAll(list);
                        list.clear();
                        if (row % 2 == 0) {
                            Collections.reverse(temp);
                        }
                        lists.add(temp);
    
                    }
                }
            }
    
            return lists;
        }

## 序列化和反序列化二叉树

解析：序列化和反序列化关键是要确定序列化方式。我么使用字符串来序列化。

用#代表空，用!分隔左右子树。

比如 1
    2 3
   4   5

使用先序遍历
序列化结果是1!2!4!###3!#5!##

反序列化先让根节点指向第一位字符，然后左子树递归进行连接，右子树

	public class Solution {
	public int index = -1;
	StringBuffer sb = new StringBuffer();
	
	String Serialize(TreeNode root) {
	       if(root == null) {
	        sb.append("#!") ;
	    }
	    else {
	        sb.append(root.val + "!");
	        Serialize(root.left);
	        Serialize(root.right);
	    }
	
	    return sb.toString();
	  }
	    TreeNode Deserialize(String str) {
	        index ++;
	        int len = str.length();
	        if(index >= len) {
	            return null;
	        }
	        String[] strr = str.split("!");
	        TreeNode node = null;
	        if(!strr[index].equals("#")) {
	            node = new TreeNode(Integer.valueOf(strr[index]));
	            node.left = Deserialize(str);
	            node.right = Deserialize(str);
	        }
	        return node;
	  }
	}

## 二叉搜索树的第k个结点

    解析：二叉搜索树的中序遍历是有序的，只需要在中序中判断数字是否在第k个位置即可。
    如果在左子树中发现了，那么递归返回该节点，如果在右子树出现，也递归返回该节点。注意必须要返回，否则结果会被递归抛弃掉。
    
    TreeNode KthNode(TreeNode pRoot, int k)
        {
            count = 0;
            return inOrderSeq(pRoot, k);
        }
        static int count = 0;
        public TreeNode inOrderSeq(TreeNode treeNode, int k) {
            if (treeNode == null) return null;
            TreeNode left = inOrderSeq(treeNode.left, k);
            if (left != null) return left;
            if (++ count == k) return treeNode;
            TreeNode right = inOrderSeq(treeNode.right, k);
            if (right != null) return right;
            return null;
        }

# 栈和队列

## 用两个队列实现栈，用两个栈实现队列。


简单说下思路

1 两个栈实现队列，要求先进先出，入队时节点先进入栈A，如果栈A满并且栈B空则把全部节点压入栈B。

出队时，如果栈B为空，那么直接把栈A节点全部压入栈B，再从栈B出栈，如果栈B不为空，则从栈B出栈。

2 两个队列实现栈，要求后进先出。入栈时，节点先加入队列A，出栈时，如果队列B不为空，则把头结点以后的节点出队并加入到队列B，然后自己出队。

如果出栈时队列B不为空，则把B头结点以后的节点移到队列A，然后出队头结点，以此类推。

## 包含min函数的栈

/**
 * 设计一个返回最小值的栈
 * 定义栈的数据结构，请在该类型中实现一个能够得到栈最小元素的min函数。
 * Created by 周杰伦 on 2017/3/22.
 */

    解析：这道题的解法也是非常巧妙的。因为每次进栈和出栈都有可能导致最小值发生改变。而我们要维护的是整个栈的最小值。
    
    如果单纯使用一个数来保存最小值，会出现一种情况，最小值出栈时，你此时的最小值只能改成栈顶元素，但这个元素不一定时最小值。
    
    所以需要一个数组来存放最小值，或者是一个栈。
    
    使用另一个栈B存放最小值，每次压栈时比较节点值和栈B顶端节点值，如果比它小则压栈，否则不压栈，这样就可以从b的栈顶到栈顶依次访问最小值，次小值。以此类推。
    
    当最小值节点出栈时，判断栈B顶部的节点和出栈节点是否相同，相同则栈B也出栈。
    
    这样就可以维护一个最小值的函数了。
    
    同理，最大值也是这样。
    
    
    public class 包含min函数的栈 {
        Stack<Integer> stack=new Stack<>();
        Stack<Integer> minstack=new Stack<>();
    
        public void push(int node) {
            if(stack.isEmpty())
            {
                stack.push(node);
                minstack.push(node);
            }
            else if(node<stack.peek()){
                stack.push(node);
                minstack.push(node);
            }
            else {
                stack.push(node);
            }
        }
        
        public void pop() {
            if(stack.isEmpty())return;
            if(stack.peek()==minstack.peek()){
               stack.pop();
               minstack.pop();
            }
            else {
                stack.pop();
            }
        }
        
        public int top() {
          return stack.peek();
        }
        
        public int min() {
            if(minstack.isEmpty())return 0;
         return minstack.peek();
        }
    }

## 栈的压入和弹出序列

        题目描述
    输入两个整数序列，第一个序列表示栈的压入顺序，请判断第二个序列是否为该栈的弹出顺序。假设压入栈的所有数字均不相等。例如序列1,2,3,4,5是某栈的压入顺序，序列4,5,3,2,1是该压栈序列对应的一个弹出序列，但4,3,5,1,2就不可能是该压栈序列的弹出序列。（注意：这两个序列的长度是相等的）
    
    解析：本题是比较抽象的，首先，根据入栈出栈的规则，我们可以建立一个栈A，用于保存压栈序列，然后压入第一个元素，比较出栈序列的第一个元素，如果不相等，继续压栈，直到二者相等，此时栈A元素出栈，然后重复上一步的操作。
    
    如果在每次压栈过程中，入栈序列已经全部入栈A但是还是找不到出栈序列的第一个元素时，则说明不是出栈序列。
    
    当栈A的元素全部压入并出栈后，如果出栈序列也出栈完毕，则满足题意。
    
    public static boolean IsPopOrder(int[] pushA, int[] popA) {
            Stack<Integer> stack = new Stack<>();
            int j = 0;
            int i = 0;
            while (i < pushA.length) {
                stack.push(pushA[i]);
                i++;
                while (!stack.empty() && stack.peek() == popA[j]) {
                    stack.pop();
                    j++;
                }
                if (i == pushA.length) {
                    if (!stack.empty()) {
                        return false;
                    } else return true;
                }
            }
            return false;
        }


# 排序和查找

## 旋转数组的最小数字

    把一个数组最开始的若干个元素搬到数组的末尾，我们称之为数组的旋转。 输入一个非递减排序的数组的一个旋转，输出旋转数组的最小元素。 例如数组{3,4,5,1,2}为{1,2,3,4,5}的一个旋转，该数组的最小值为1。 NOTE：给出的所有元素都大于0，若数组大小为0，请返回0。
    
    解析：这题的思路很巧妙，如果直接遍历复杂度为O（N），但是使用二分查找可以加快速度，因为两边的数组都是递增的最小值一定在两边数组的边缘，于是通过二分查找，逐渐缩短左右指针的距离，知道左指针和右指针只差一步，那么右指针所在的数就是最小值了。
    复杂度是O（logN)
    
    //这段代码忽略了三者相等的情况
    public int minNumberInRotateArray(int [] array) {
            if (array.length == 0) return 0;
            if (array.length == 1) return array[0];
            int min = 0;
    
            int left = 0, right = array.length - 1;
            //只有左边值大于右边值时，最小值才可能出现在中间
            while (array[left] > array[right]) {
                int mid = (left + right)/2;
                if (right - left == 1) {
                    min = array[right];
                    break;
                }
                //如果左半部分递增，则最小值在右侧
                if (array[left] < array[mid]) {
                    left = mid;
                }
                //如果右半部分递增，则最小值在左侧。
                //由于左边值比右边值大，所以两种情况不会同时发生
                else if (array[right] > array[mid]) {
                    right = mid ;
                }
            }
            return array[min];
    
        }

 

        注意：但是当arr[left] = arr[right] = arr[min]时。三个数都相等无法确定最小值，此时只能遍历。

# 递归

## 斐波那契数列

1递归做法

2记忆搜索，用数组存放使用过的元素。

3DP，本题中dp就是记忆化搜索

## 青蛙跳台阶

一次跳两步或者跳一步，问一共多少种跳法到达n级，所以和斐波那契数列是一样的。

## 变态跳台阶

一次跳1到n步，问一共几种跳法，这题是找数字规律的，一共有2^(n-1)种方法

## 矩形覆盖

和上题一样，也是找规律，答案也是2^(n-1)

# 位运算

## 二进制中1的个数

 * Created by 周杰伦 on 2018/6/29.
 * 题目描述
 * 输入一个整数，输出该数二进制表示中1的个数。其中负数用补码表示。

    解析:
    1 循环右移数字n，每次判断最低位是否为1，但是可能会导致死循环。
    
    2 使用数字a = 1和n相与，a每次左移一位，再与n相与得到次低位，最多循环32次，当数字1左移32次也会等于0，所以结束循环。
    
    3 非常奇葩的做法，把一个整数减去1，再与原整数相与，会把最右边的一个1变成0，于是统计可以完成该操作的次数即可知道有多少1了。
    
        public class 二进制中1的个数 {
            public static int NumberOf1(int n) {
                int count = 0;
                while (n != 0) {
                    ++count;
                    n = (n - 1) & n;
                }
                return count;
            }
        }
    
## 数组中只出现一次的数字

题目描述
一个整型数组里除了一个数字之外，其他的数字都出现了两次。请写程序找出这一个只出现一次的数字。

解析：左神称之为神仙题。

利用位运算的异或操作^。
由于a^a = 0,0^b=b，所以。所有数执行异或操作，结果就是只出现一次的数。

## 不用加减乘除做加法

解析：不用加减乘，那么只能用二进制了。

两个数a和b，如果不考虑进位，则0 + 1 = 1,1 + 1 = 0，0 + 0 = 0，这就相当于异或操作。
如果考虑进位，则只有1 + 1有进位，所以使用相与左移的方法得到每一位的进位值，再通过异或操作和原来的数相加。当没有进位值的时候，运算结束。  
    public static int Add(int num1,int num2) {
        if( num2 == 0 )return num1;
        if( num1 == 0 )return num2;

        int temp = num2;
        while(num2!=0) {
            temp = num1 ^num2;
            num2 = (num1 & num2)<<1;
            num1 = temp;
        }
        return num1;
    }

# 回溯和DFS

## 矩阵中的路径

题目描述
请设计一个函数，用来判断在一个矩阵中是否存在一条包含某字符串所有字符的路径。路径可以从矩阵中的任意一个格子开始，每一步可以在矩阵中向左，向右，向上，向下移动一个格子。如果一条路径经过了矩阵中的某一个格子，则之后不能再次进入这个格子。 例如 a b c e s f c s a d e e 这样的3 X 4 矩阵中包含一条字符串"bcced"的路径，但是矩阵中不包含"abcb"路径，因为字符串的第一个字符b占据了矩阵中的第一行第二个格子之后，路径不能再次进入该格子。

解析：回溯法也就是特殊的dfs，需要找到所有的路径，所以每当到达边界条件或抵达目标时，递归返回，由于需要保存路径中的字母，所以递归返回时需要删除路径最后的节点，来保证路径合法。不过本题只有一个解，所以找到即可返回。

    public class 矩阵中的路径 {
        public static void main(String[] args) {
            char[][]arr = {{'a','b','c','e'},{'s','f','c','s'},{'a','d','e','e'}};
            char []str = {'b','c','c','e','d'};
            System.out.println(hasPath(arr, arr.length, arr[0].length, str));
        }
        static int flag = 0;
        public static boolean hasPath(char[][] matrix, int rows, int cols, char[] str)
        {
            int [][]visit = new int[rows][cols];
            StringBuilder sb = new StringBuilder();
            for (int i = 0;i < rows;i ++) {
                for (int j = 0;j < cols;j ++) {
                    if (matrix[i][j] == str[0]) {
                        visit[i][j] = 1;
                        sb.append(str[0]);
                        dfs(matrix, i, j, visit, str, 1, sb);
                        visit[i][j] = 0;
                        sb.deleteCharAt(sb.length() - 1);
                    }
                }
            }
            return flag == 1;
        }
        public static void dfs(char [][]matrix, int row, int col, int [][]visit, char []str, int cur, StringBuilder sb) {
            if (sb.length() == str.length) {
    //            System.out.println(sb.toString());
                flag = 1;
                return;
            }
    
            int [][]pos = {{1,0},{-1,0},{0,1},{0,-1}};
            for (int i = 0;i < pos.length;i ++) {
                int x = row + pos[i][0];
                int y = col + pos[i][1];
                if (x >= matrix.length || x < 0 || y >= matrix[0].length || y < 0) {
                    continue;
                }
                if (visit[x][y] == 0 && matrix[x][y] == str[cur]) {
                    sb.append(matrix[x][y]);
                    visit[x][y] = 1;
                    dfs(matrix, x, y, visit, str, cur + 1, sb);
                    sb.deleteCharAt(sb.length() - 1);
                    visit[x][y] = 0;
                }
            }
        }


​    
## 机器人的运动范围

题目描述
地上有一个m行和n列的方格。一个机器人从坐标0,0的格子开始移动，每一次只能向左，右，上，下四个方向移动一格，但是不能进入行坐标和列坐标的数位之和大于k的格子。 例如，当k为18时，机器人能够进入方格（35,37），因为3+5+3+7 = 18。但是，它不能进入方格（35,38），因为3+5+3+8 = 19。请问该机器人能够达到多少个格子？

 解析：这是一个可达性问题，使用dfs方法，走到的每一格标记为走过，走到无路可走时就是最终的结果。每次都有四个方向可以选择，所以写四个递归即可。

    public class Solution {
        static int count = 0;
        public static int movingCount(int threshold, int rows, int cols)
        {
            count = 0;
            int [][]visit = new int[rows][cols];
            dfs(0, 0, visit, threshold);
            return count;
        }
    
        public static void dfs(int row, int col, int[][]visit, int k) {
            if (row >= visit.length || row < 0 || col >= visit[0].length || col < 0) {
                return;
            }
            if (sum(row) + sum(col) > k) {
                return;
            }
    
            if (visit[row][col] == 1){
                return;
            }
    
            visit[row][col] = 1;
            count ++;
            dfs(row + 1,col,visit, k);
            dfs(row - 1,col,visit, k);
            dfs(row,col + 1,visit, k);
            dfs(row,col - 1,visit, k);
    
        }
    
        public static int sum(int num) {
            String s = String.valueOf(num);
            int sum = 0;
            for (int i = 0;i < s.length();i ++) {
                sum += Integer.valueOf(s.substring(i, i + 1));
            }
            return sum;
        }
    }

