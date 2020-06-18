---
title: 代币的最佳组合
date: 2018-08-10 20:26:30
tags:
    - 算法
categories: 算法
---

上一篇写了关于下单优惠的计算,项目迭代了几期,产品又提出要在下单后支付前还要有新的优惠措施,以一种不同面值的代币形式.买家可以选择不同面值的组合减免支付金额.

#### 问题描述

当用户下单后进入支付页面,系统优先计算出最划算的组合,供给用户选择.故实际付款金额=下单金额-代币组合值,代币不限数量,所以组合会有很多组合.例如有10,10,45,50,100的面值的代币,下单金额为75,最优组合为10,10,50.

<!-- more -->
#### 解决思路

##### 全排列方式

既然是多种组合,当我们使用迭代排列出全部组合,然后找出最优的解,但是这样的暴力穷举法,时间复杂度和空间复杂度都是很大,移动端性能本身就较弱,这样计算量和存储无法负担,并且计算时间长,没有现实意义.在测试用例中当有24个数字时, fullCombination 方法耗时平均15s左右,并且这是在 PC 执行的结果.
``` java
    public List<List<Coin>> fullCombination(List<Coin> lstSource) {
        int n = lstSource.size();
        int max = 1 << n;//2的n次方
        List<List<Coin>> lstResult = new ArrayList<>();
        for (int i = 0; i < max; i++) {
            List<Coin> lstTemp = new ArrayList<>();
            for (int j = 0; j < n; j++) {
                //i除以2的j次方
                if ((i >> j & 1) > 0) {
                    lstTemp.add(lstSource.get(j));
                }
            }
            lstResult.add(lstTemp);
        }
        lstResult.remove(0);
        return lstResult;
    }
```
##### 回溯算法

我们可以把这个问题抽象成:给定一个数组 candidates 和一个目标数 target ，找出 candidates 中所有可以使数字和为 target 的组合.candidates 中的每个数字在每个组合中只能使用一次.这个就是 LeetCode 中的组合总和II的题目,我们的需求完全一样,只需要小调整就可以使用,并且在时间复杂度O(n!)内,实际应用中可完全接受.使用上面的测试用例,得到最后的最优组合只需要20ms左右.

``` java 
public class Coin {
    public String id;
    /**
     * 面值
     */
    public double denomination;
}
    /**
     * 获取最优组合
     *
     * @param target 下单金额
     * @param coins  所有代币
     * @return 最优组合中的代币
     */
    private List<Coin> getBestCombination(double target, List<Coin> coins) {
        List<Coin> result = new ArrayList<>(5);
        List<ArrayList<Coin>> combinations = new ArrayList<>(10);
        //所有代币组合
        combinations = combinationSum(coins, target, combinations);

        TreeMap<Integer, List<Coin>> lastResult = new TreeMap<>();
        //根据组合中的代币 计算出总和按升序排列
        for (List<Coin> arrays : combinations) {
            int temp = 0;
            for (Coin c : arrays) {
                temp += c.denomination;
            }
            lastResult.put(temp, arrays);
        }
        //获取最后一个组合即为最优组合
        result.addAll(lastResult.lastEntry().getValue());
        return result;
    }

    private List<ArrayList<Coin>> combinationSum(List<Coin> coins,
                                                 double target,
                                                 List<ArrayList<Coin>> result) {
        List<ArrayList<Coin>> res = new ArrayList<>(10);
        if (coins == null || coins.size() == 0) return new ArrayList<>(1);
        List<Coin> realCoins = new ArrayList<>(coins.size());
        for (Coin c : coins) {
            if (c.denomination <= target) {
                realCoins.add(c);
            }
        }
        Collections.sort(realCoins, new Comparator<Coin>() {
            @Override
            public int compare(Coin c1, Coin c2) {
                return Double.compare(c1.denomination, c2.denomination);
            }
        });
        return dfs(realCoins, 0, target, new ArrayList<Coin>(10), res, result);
    }

    private List<ArrayList<Coin>> dfs(List<Coin> candidates,
                                      int startIndex,
                                      double target,
                                      List<Coin> combination,
                                      List<ArrayList<Coin>> res, List<ArrayList<Coin>> result) {
        //存储所有组合结果
        result.add(new ArrayList<>(combination));

        if (target == 0) {
            res.add(new ArrayList<>(combination));
            return result;
        }

        for (int i = startIndex, size = candidates.size(); i < size; i++) {
            if (i != startIndex && candidates.get(i).denomination == candidates.get(i - 1).denomination) {
                continue;
            }
            if (target < candidates.get(i).denomination) {
                break;
            }
            combination.add(candidates.get(i));
            dfs(candidates, i + 1, target - candidates.get(i).denomination, combination, res, result);
            combination.remove(combination.size() - 1);
        }
        return result;
    }
```
[项目地址](https://github.com/Thewhitelight/Tinder/blob/master/app/src/test/java/cn/libery/tinder/CoinTest.java) 包含两种方法的所有代码和测试用例.

#### 参考
[组合总和 II LeetCode](https://leetcode-cn.com/problems/combination-sum-ii/description/)
[浅谈回溯与深度优先搜索](https://blog.csdn.net/James_T_Kirk/article/details/76895209)
[组合总和 II 解法](https://www.jiuzhang.com/solution/combination-sum-ii/)



