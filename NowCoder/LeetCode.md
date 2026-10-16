## 数组

### NO.977

双指针思路  O(n) 

```c++
class Solution {
public:
    vector<int> sortedSquares(vector<int>& nums) {
        int i = 0;
        int size = nums.size();
        while(i < size){
            nums[i] = nums[i] * nums[i];
            i++;
        }
        int left = 0;
        int right = size - 1;
        i = size - 1; 
        vector<int> ans(size);
        while(left <= right){
            if(nums[left] >= nums[right]){
                ans[i] = nums[left];
                left++;
            }
            else{
                ans[i] = nums[right];
                right--;
            }
            i--;
        }
        return ans;
    }
};
```

### NO.209

暴力解法：

```c++
class Solution {
public:
    int minSubArrayLen(int target, vector<int>& nums) {
        for(int windowsize=1; windowsize<=nums.size(); windowsize++){
            for(int index = 0; index < nums.size()-windowsize+1; index++){
                int sum = 0;
                for(int i = index; i < windowsize+index; i++){
                    sum += nums[i];
                } 
                if (sum>=target){
                    return windowsize;
                }
            } 
        }
        return 0;
    }
};
```

滑动窗口：

```c++
class Solution {
public:
    int minSubArrayLen(int target, vector<int>& nums) {
        int size = nums.size();
        int sum = 0;
        int min_len = size + 1;

        int i = 0;
        int j = 0;
        while(j < size){
            sum += nums[j];
            int temp_len = j - i + 1;
            while(sum >= target){
                if(temp_len < min_len){
                    min_len = temp_len;
                }
                sum -= nums[i];
                i = i + 1;
                temp_len--;
            }
            j++;
        }
        if (min_len == size + 1) return 0;
        else return min_len;
    }
};
```

### NO.76

滑动窗口+哈希表

- 两个哈希表：一个存储需求字符，一个存储窗口字符