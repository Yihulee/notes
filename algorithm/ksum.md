# `ksum`

文章来自 [http://www.sigmainfy.com/blog/summary-of-ksum-problems.html](http://www.sigmainfy.com/blog/summary-of-ksum-problems.html)

做过`leetcode`的人都知道，里面有`2sum`, `3sum`(`closest`), `4sum`等问题，这些也是面试里面经典的问题，考察是否能够合理利用排序这个性质，一步一步得到高效的算法。经过总结，本人觉得这些问题都可以使用一个通用的`K` `sum`求和问题加以概括消化，这里我们先直接给出`K` `Sum`的问题描述和算法（递归解法）, 然后将这个一般性的方法套用到具体的`K`, 比如`leetcode`中的`2Sum`, `3Sum`, `4Sum`问题。同时我们也给出另一种哈希算法的讨论。

### `leetcode`求和问题描述 (`K` `sum` `problem`)：

`K sum`的求和问题一般是这样子描述的：给你一组`N`个数字（比如 `vector` `num`), 然后给你一个常数（比如 `int` `target`) ，我们的`goal`是在这一堆数里面找到`K`个数字，使得这`K`个数字的和等于`target`。

### 注意事项 (`constraints`):

注意这一组数字可能有重复项：比如 `1 1 2 3` , 求`3sum`, 然后 `target  = 6`, 你搜的时候可能会得到 两组`1 2 3`,` 1 2 3`，`1` 来自第一个`1`或者第二个`1`, 但是结果其实只有一组，所以最后结果要去重。

### `K` `Sum`求解方法，适用`leetcode` `2Sum`, `3Sum`, `4Sum`：

**方法一：** 暴力，就是枚举所有的`K-subset`, 那么这样的复杂度就是 从`N`选出`K`个，复杂度是`O(N^K)`

**方法二：** 排序，这个算法可以考虑最简单的`case`, `2sum`，这是个经典问题，方法就是先排序，然后利用头尾指针找到两个数使得他们的和等于`target`, 这个`2sum`算法网上一搜就有，这里不赘述了，给出`2sum`的核心代码：

```c++
//2 sum
int i = starting; // 头指针
int j = num.size() - 1; // 尾指针
while(i < j) {
    int sum = num[i] + num[j];
        if(sum == target) {
            store num[i] and num[j] somewhere;
                if(we need only one such pair of numbers)
                    break;
                 otherwise
                 do ++i, --j;
        }
        else if(sum < target)
            ++i;
        else
            --j;
}
```

`2sum`的算法复杂度是`O(N log N)` 因为排序用了`N log N`以及头尾指针的搜索是线性的，所以总体是`O(N log N)`，好了现在考虑`3sum`, 有了`2sum`其实 3sum 就不难了，这样想：先取出一个数，那么我只要在剩下的数字里面找到两个数字使得他们的和等于 (`target` – 那个取出的数）就可以了吧。所以`3sum`就退化成了`2sum`, 取出一个数字，这样的数字有`N`个，所以`3sum`的算法复杂度就是`O(N^2 )`, 注意这里复杂度是`N`平方，因为你排序只需要排一次，后面的工作都是取出一个数字，然后找剩下的两个数字，找两个数字是`2sum`用头尾指针线性扫，这里很容易错误的将复杂度算成`O(N^2 log N)`，这个是不对的。我们继续的话`4sum`也就可以退化成`3sum`问题，那么以此类推，`K-sum`一步一步退化，最后也就是解决一个`2sum`的问题，`K sum`的复杂度是`O(n^(K-1))`。 这个界好像是最好的界了，也就是`K-sum`问题最好也就能做到`O(n^(K-1))`复杂度，之前有看到过有人说可以严格数学证明，这里就不深入研究了。

**更新：** 感谢网友`Hatch`提供他的`K Sum`源代码，经供参考：

```c++
class Solution {
public:
	vector<vector> findZeroSumInSortedArr(vector &num, int begin, int count, int target)
	{
		vector ret;
		vector tuple;
		set visited;
		if (count == 2)
		{
			int i = begin, j = num.size()-1;
			while (i < j)
			{
				int sum = num[i] + num[j];
				if (sum == target && visited.find(num[i]) == visited.end())
				{
					tuple.clear();
					visited.insert(num[i]);
					visited.insert(num[j]);
					tuple.push_back(num[i]);
					tuple.push_back(num[j]);
					ret.push_back(tuple);
					i++; j–;
				}
				else if (sum < target)
				{
					i++;
				}
				else
				{
					j–;
				}
			}
		}
		else
		{
			for (int i=begin; i < num.size(); i++)
			{
				if (visited.find(num[i]) == visited.end())
				{
					visited.insert(num[i]);
					vector subRet = findZeroSumInSortedArr(num, i+1, count-1, target-num[i]);
					if (!subRet.empty())
					{
						for (int j=0; j < subRet.size(); j++)
						{
							subRet[j].insert(subRet[j].begin(), num[i]);
						}

						ret.insert(ret.end(), subRet.begin(), subRet.end());
					}
				}
			}
		}

		return ret;
	}

	vector threeSum(vector &num) {
		sort(num.begin(), num.end());
		return findZeroSumInSortedArr(num, 0, 3, 0);
	}

	vector fourSum(vector &num, int target) {
		sort(num.begin(), num.end());
		return findZeroSumInSortedArr(num, 0, 4, target);
	}
}
```

### `K Sum` (`2Sum`, `3Sum`, `4Sum`) 算法优化 (`Optimization`)：

这里讲两点，第一，注意比如`3sum`的时候，先整体排一次序，然后枚举第三个数字的时候不需要重复， 比如排好序以后的数字是 `a b c d e f`, 那么第一次枚举`a`, 在剩下的`b c d e f`中进行`2 sum`, 完了以后第二次枚举`b`, 只需要在 `c d e f`中进行`2sum`好了，而不是在`a c d e f`中进行`2sum`, 这个大家可以自己体会一下，想通了还是挺有帮助的。第二，`K Sum`可以写一个递归程序很优雅的解决，具体大家可以自己试一试。写递归的时候注意不要重复排序就行了。

### `K Sum` (`2Sum`, `3Sum`, `4Sum`) 算法之`3sum`源代码（不使用`std::set`) 和相关开放问题讨论：

因为已经收到好几个网友的邮件需要`3sum`的源代码，那么还是贴一下吧，下面的代码是可以通过`leetcode OJ`的代码（又重新写了一遍，于`Jan, 11, 2014 Accepted`), 就当是`K sum`的完整的一个`case study`吧，顺便解释一下上面的排序这个注意点，同时我也有关于结果去重的问题可以和大家讨论一下，也请大家集思广益，发表意见，首先看源代码如下：

```java
class Solution {
public:
    vector threeSum(vector &num) {
		vector vecResult;
		if(num.size() < 3)
			return vecResult;

		vector vecTriple(3, 0);
		sort(num.begin(), num.end());
		int iCurrentValue = num[0];
		int iCount = num.size() - 2; // (1) trick 1
		for(int i = 0; i &lt; iCount; ++i) {
			if(i && num[i] == iCurrentValue) { // (2) trick 2: trying to avoid repeating triples
				continue;
			}
			// do 2 sum
			vecTriple[0] = num[i];
			int j = i + 1;
			int k = num.size() - 1;
			while(j < k) {
				int iSum = num[j] + num[k];
				if(iSum + vecTriple[0] == 0) {
					vecTriple[1] = num[j];
					vecTriple[2] = num[k];
					vecResult.push_back(vecTriple); // copy constructor
					++j; // 向中间靠拢
					--k;
				}
				else if(iSum + vecTriple[0] < 0)
					++j;
				else
					--k;
			}
			iCurrentValue = num[i];
		}
        // trick 3: indeed remove all repeated triplets
        // trick 4: already sorted, no need to sort the triplets at all, think about why?
		vector<vector<int>>::iterator it = unique(vecResult.begin(), vecResult.end());
		vecResult.resize(distance(vecResult.begin(), it));
		return vecResult;
    }
};
```

首先呢，在`K Sum`问题中都有个结果去重的问题，前文也说了，如果输入中就有重复元素的话，最后结果都需要去重，去重有好几个办法，可以利用`std::set`的性质（如`leetcode`上`3sum`的文章，但是他那个文章的问题是，`set`没用好，导致最终复杂度其实是`O(N^2 * log N)`, 而非真正的`O(N^2)` ), 可以利用排序（如本文的方法）等，去重本身不难，难的是不利用任何排序或者`std::set`直接得到没有重复的 triplet 结果集。本人试图通过已经排好序这个性质来做到这一点（试图不用`trick 3`和`4`下面的两条语句）,　但是经过验证这样的代码（没有`trick 3`,` 4`下面的两行代码，直接`return vecResult`) 也不能保证结果没有重复，于是不得不加上了`trick 3`, `4`，还是需要通过在结果集上进一步去重.**笔者对于这个问题一直没有很好的想法，希望这里的代码能抛砖引玉，大家也讨论一下有没有办法，或者利用排序的性质或者利用其它方法，直接得到没有重复元素的 triplet 结果集，不需要去重这个步骤.**

那么还是解释一下源代码里面有四个`trick`, **以及笔者试图不利用任何`std::set`或者排序而做到去重的想法**. 第一个无关紧要顺带的小`trick 1`, 是说我们排好序以后，只需要检测到倒数第三个数字就行了，因为剩下的只有一种`triplet` 由最后三个数字组成。接下来三个`trick`都是和排序以及最后结果的去重问题有关的，我一起说。

笔者为了达到不需要在最后的结果集做额外的去重，尝试了以下努力：首先对输入数组整体排序，然后使用之前提到的`3sum`的算法，每次按照顺序先定下`triplet`的第一个数字，然后在数组后面寻找`2sum`, 由于已经排好序，为了防止重复，我们要保证`triplet`的第一个数字没有重复，举个例子，` -3, – 3, 2, 1`, 那么第二个`-3`不应该再被选为我们的第一个数字了， **因为在第一个`-3`定下来寻找`2 sum`的时候，我们一定已经找到了所有以`-3`为第一个数字的 triplet(trick 2). ** 但是这样做尽管可以避免一部分的重复，但是还有另一种重复无法避免：`-3, -3, -3, 6`, 那么在定下第一个`-3`的时候，我们已经有两组重复`triplet <-3, -3, 6>`， 如何在不使用`std::set`的情况下避免这类重复，笔者至今没有很好的想法. **大家有和建议？望不吝赐教！**

**更新： **感谢网友`stayshan`的留言提醒，根据他的留言，不用在最后再去重。于是可以把`trick 3, 4`下面的两行代码去掉，然后把`while`里面的`copy constructor`这条语句加上是否和前一个元素重复的判断变成下面的代码就行了。

这样的做法当然比我上面的代码更加优雅，虽然本质其实是一样的，只不过去重的阶段变化了，进一步的，我想探讨的是，我们能不能通过”不产生任何重复的`triplet`”的方法直接得到没有重复的`triplet`集合？ 网友`stayshan`提到的方法其实还是可能生成重复的`triplet`, 然后通过和已有的`triple`t 集合判断去重， **笔者在这里试图所做的尝试更加确切的讲是想找到一种方法，可以保证不生成重复的`triplet`. 现有的方法似乎都是`post-processing`, i.e., 生成了重复的`triplet`以后进行去重。笔者想在这里探讨从而找到一种我觉得可以叫他`pre-processing`的方法，能够通过一定的性质（可能是排序的性质等）保证不会生成`triplet`, 从而达到不需任何去重的后处理 (`post-processing`) 手段。感觉笔者抛出的砖已经引出了挺好的思路了啊，太好了，大家有啥更好的建议，还请指教啊 ：） **

```java
class Solution {
public:
    vector threeSum(vector &num) {
		// same as above
                // ...
		for(int i = 0; i &lt; iCount; ++i) {
			// same as above
                        // ...
			while(j &lt; k) {
				int iSum = num[j] + num[k];
				if(iSum + vecTriple[0] == 0) {
					vecTriple[1] = num[j];
					vecTriple[2] = num[k];
					if(vecResult.size() == 0 || vecTriple != vecResult[vecResult.size() - 1])
						vecResult.push_back(vecTriple); // copy constructor
					++j;
					--k;
				}
				else if(iSum + vecTriple[0] &lt; 0)
					++j;
				else
					--k;
			}
			iCurrentValue = num[i];
		}
		return vecResult;
    }
 };
```

### Hash 解法 (`Other`)：

其实比如`2sum`还是有线性解法的，就是用`hashmap`, 这样你`check`某个值存在不存在就是常数时间，那么给定一个`sum`, 只要线性扫描，对每一个`number`判断`sum – num`存在不存在就可以了。注意这个算法对有重复元素的序列也是适用的。比如 `2 3 3 4` 那么`hashtable`可以使 `hash(2) = 1; hash(3) = 1, hash(4) =1`其他都是`0`,  那么`check`的时候，扫到两次`3`都是`check sum – 3`在不在`hashtable`中，注意最后返回所有符合的`pair`的时候也还是要去重。这样子推广的话` 3sum` 其实也有`O(N^2)`的类似`hash`算法，这点和之前是没有提高的，但是`4sum`就会有更快的一个算法。

### `4sum`的`hash`算法：

`O(N^2)`把所有`pair`存入`hash`表，并且每个`hash`值下面可以跟一个`list`做成`map`， `map[hashvalue] = list`，每个`list`中的元素就是一个`pair`, 这个`pair`的和就是这个`hash`值，那么接下来求`4sum`就变成了在所有的 pair value 中求 `2sum`，这个就成了线性算法了，注意这里的线性又是针对`pair`数量`(N^2)`的线性，所以整体上这个算法是`O(N^2)`，而且因为我们挂了`list`, 所以只要符合`4sum`的我们都可以找到对应的是哪四个数字。

到这里为止有人提出这个算法不对 （感谢 Jun 提出这点！! `See the comment below`), 因为这里的算法似乎无法检查取出来的四个数字是否有重复的，也就是说在转换成`2sum`问题得到的那些个`pair`中，有可能会有重复元素，比如说原来数组中的第一个元素其实是重复了两次才使得`4 sum`满足要求，那么这样得到的四元组（四个数字的和等于给定的值）, 其实只有三个原数组元素，其中第一个元素用了两次，那么这样就不对了。如果仅从我上面的描述来看，确实是没有办法检查重复的，但是仔细想想我们只要在`map`中存`pair`的的时候记录下的不是原数组对应的值，而是原数组的`id`, 就可以避免这个问题了。更加具体的，`map`[hashvalue] = list, 每个 list 的元素就是一个`pair`, 这个`pair<int, int>` 中的`pair`是原来的`array id`, 使得这两个`id`对应到元素组中的元素值的和就是这个 hash 值。那么问题就没有了，我们在转换成的`2sum`寻找所有`pair value`的`2sum`的同时要检查最后得到的四元组`<id1, id2, id3, id4>`没有重复`id`. 这样问题就解决了。

更新： 关于`4Sum`的`Hash`解法，感谢网友`Tenos`和`hahaer`的评论，笔者再三思考，思来想去》_《对于`hahaer`提出的所有元素都是`0`, 而且`Target`也是`0`的这个`case`, 我想问题可能在这里。

首先如果要找出所有唯一的四元组`(id1, id2, id3, id4)`也就是`id`级别的四元组，那么时间复杂度只能是`O(N^4)`. 推理如下：如果要找到所有的唯一的四元组`(id1, id2, id3, id4)`的话，是一定要`O(N^4)`时间的，因为在这个`case`里面，就是一个组合问题，在`N`个`id`里面任意取出`4`个不同的`id`, 都是符合我们条件的四元组，光是这样，答案就有 `O(N^4)`个，`N`个里面取四个的组合种数。

可是！如果大家再去看看`leetcode`的题目的话，其实题目要求是返回元素组成的四元组（而不是要求 id 组成的四元组唯一）, 也就是元素级别的四元组（参考网友`Jun`和`AmazingCaddy`和我在评论中的讨论）在这个`case`中，返回`(0, 0, 0, 0)`就好了，而不是返回`(id1， id2, id3, id4)`也就是不用去管`id`的问题。如果是这样的话我们就不需要比较`id`了，利用`set`之类的`post-processing`的方法是可以得到唯一的`(0, 0, 0, 0)`的。

还是抛砖引玉吧，如果大家在这个问题上还有什么想法，还请留言指点。

### 结束语：

这篇文章主要想从一般的`K sum`问题的角度总结那些比较经典的求和问题比如`leetcode`里面的`2sum`, `3sum(closest)`,` 4sum`等问题， 文章先直接给出`K Sum`的问题描述和算法（递归解法）, 然后将这个一般性的方法套用到具体的`K`, 比如`leetcode`中的`2Sum`, `3Sum`, `4Sum`问题。同时我们也给出另一种哈希算法的讨论. 那么这篇文章基本上还是自己想到什么写什么，有疏忽不对的地方请大家指正，也欢迎留言讨论，如果需要源代码，请留言或者发邮件到 sigmainfy.tech@gmail.com
