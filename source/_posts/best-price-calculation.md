---
title: 根据订单金额匹配最划算的优惠券与定金
date: 2018-08-09 21:47:26
tags:
    - 算法
categories: 算法
---

在电商平台通常会有一些优惠措施,比如以优惠券、定金或者代币之类的形式在下单时给予用户直观的减免金额.一般用户领取店铺优惠券,优惠券有不同的使用条件,满足条件时方可使用.定金为用户自己向店铺购买的形式,定金在特定的时间或者针对某个商品可以膨胀使用,膨胀出多余的部分相当商家让利,可以优惠购买.代币为商家以不同面值发放给用户,以抵扣下单金额.

#### 问题描述

假设某个店铺有好多不同使用条件的优惠券比如满20减去5,满20减10,每满20减5,每满20减21(其实我也不知道这个的意义什么,但是产品说就要发这样的券),然后用户还分别购买了20,30,60的定金,当用户下单金额为200元时,用户自己就难以选择,无法立即计算出支付最少的钱购买此单商品,所以需要我们计算匹配出一个最划算的组合给用户.当然优惠券和定金只能各选一个.

#### 解决思路

作为用户来讲,当然优先使用优惠券,因为优惠券为商家赠与而定金时自己购买,故支付金额=订单金额-优惠券的优惠金额(因为包含每满类型,所以不一是优惠券面值)-定金.那么大概思路就是定金和优惠券的组合,相当于两层for循环,然后按照这个公式算出支付金额大于等于零的那个值就可以了.思路时挺简单,但是优惠券的逻辑就有点复杂,以为优惠券自身就有好多种组合,例如满10减5,满10减8,每满10减8,满5减8,每满5减8,所以优惠券的顺序应该根据是否每满、优惠满足金额、优惠金额排序.

<!-- more -->

#### 具体方案

先定义好优惠券、定金和最优组合的模型
``` java
public class Coupon {
    /**
     * 优惠券id
     */
    public String id;
    /**
     * 优惠券满足金额
     */
    public double spending;
    /**
     * 是否每满 0为否
     */
    public int every;
    /**
     * 优惠金额
     */
    public double discount;
}

public class Booking {
    /**
     * 定金Id
     */
    public String id;
    /**
     * 定金金额
     */
    public double amount;
}

public class BestCombination {
    /**
     * 使用定金金额
     */
    public double bookingAmount;
    /**
     * 使用优惠券金额
     */
    public double couponAmount;
    /**
     * 使用的定金
     */
    public Booking booking;
    /**
     * 使用的优惠券
     */
    public Coupon coupon;

    public BestCombination(Booking booking, Coupon coupon, double bookingAmount, double couponAmount) {
        this.booking = booking;
        this.coupon = coupon;
        this.bookingAmount = bookingAmount;
        this.couponAmount = couponAmount;
    }
}
```
然后先对优惠券和定金列表排序
``` java
private List<Coupon> initCoupon(double totalPrice, List<Coupon> coupons) {
        List<Coupon> result = new ArrayList<>();
        if (coupons.size() > 0) {
            for (Coupon b : coupons) {
                //排除掉面值大于订单金额的券
                if (b.spending <= totalPrice) {
                    result.add(b);
                }
            }
        }

        //依次根据是否每满、优惠满足金额、优惠金额排序
        Collections.sort(result, new Comparator<Coupon>() {
            @Override
            public int compare(Coupon o1, Coupon o2) {
                //每满券在前
                int compareEvery = Integer.compare(o2.every, o1.every);
                if (compareEvery == 0) {
                    //面值小的在前,为了近可能的不让用户使用定金
                    int compareSpending = Double.compare(o1.spending, o2.spending);
                    if (compareSpending == 0) {
                        return Double.compare(o1.discount, o2.discount);
                    } else {
                        return compareSpending;
                    }
                } else {
                    return compareEvery;
                }
            }
        });

        //用户可以选择不适用任何优惠券,所以需要添加此
        Coupon emptyCoupon = new Coupon();
        emptyCoupon.discount = 0;
        emptyCoupon.spending = 0;
        emptyCoupon.id = "-1";
        emptyCoupon.every = 0;
        result.add(emptyCoupon);
        return result;
    }

    //定金只需要加入不使用定金的选项
    private List<Booking> initBooking(List<Booking> bookings) {
        List<Booking> result = new ArrayList<>();
        if (bookings.size() > 0) {
            result.addAll(bookings);
        }

        Booking emptyBooking = new Booking();
        emptyBooking.amount = 0;
        emptyBooking.id = "-1";
        result.add(emptyBooking);
        return result;
    }

```
最后就可以根据订单金额、优惠券和定金计算出最优的组合
``` java
  /**
     * 计算最划算的组合
     *
     * @param totalPrice 订单金额
     * @param coupons    排好序的优惠券列表
     * @param bookings   定金列表
     * @return 最优组合
     */
    public static BestCombination calculatorBestCombination(double totalPrice, List<Coupon> coupons, List<Booking> bookings) {
        //TreeMap已对key进行排序,用来装填所有的组合结果,key为支付金额
        SortedMap<Double, BestCombination> map = new TreeMap<>();
        double couponPrice;

        for (Coupon c : coupons) {
            for (Booking b : bookings) {
                double tempPrice = totalPrice;
                if (c.every == 1) {
                    //每满券相当于多张券 张数*优惠金额
                    couponPrice = Math.floor(tempPrice / c.spending) * c.discount;
                } else {
                    couponPrice = c.discount;
                }
                //订单金额-所有优惠金额
                tempPrice = tempPrice - couponPrice;

                //支付金额
                tempPrice = tempPrice - b.amount;
                //当定金面值为正值且支付金额为负值时，排除此价格进入最优价格队列
                if (!(tempPrice < 0 && b.amount > 0)) {
                    if (map.containsKey(tempPrice)) {
                        //当支付金额相同时,如果组合内的定金大于将要插入的定金金额(意味着所使用的优惠券小)，则替换原有的组合
                        BestCombination best = map.get(tempPrice);
                        if (best.bookingAmount >= b.amount) {
                            map.put(tempPrice, new BestCombination(b, c, b.amount, couponPrice));
                        }
                    } else {
                        map.put(tempPrice, new BestCombination(b, c, b.amount, couponPrice));
                    }
                }

            }
        }
        BestCombination bestCombination;
        if (map.size() <= 0) {
            //组合队列为空，则无最优组合
            bestCombination = null;
        } else {
            if (map.firstKey() >= 0) {
                //当组合队列第一个值大于零，则取第一个，此时为最优惠
                bestCombination = map.get(map.firstKey());
            } else {
                double key = 0;
                for (Map.Entry<Double, BestCombination> e : map.entrySet()) {
                    System.out.println(e.getKey() + " " + e.getValue().toString());
                    if (e.getValue().booking.amount > 0) {
                        if (e.getKey() >= 0) {
                            break;
                        } else {
                            //定金大于零则说明优惠券使用了每满且类似10减20的类型 当支付金额小于零时
                            key = e.getKey();
                        }
                    } else {
                        if (e.getKey() > 0) {
                            break;
                        } else {
                            //定金等于0 且支付金额小于等于零
                            key = e.getKey();
                        }
                    }
                }
                bestCombination = map.get(key);
            }
        }
        return bestCombination;
    }
```
计算最划算的组合,重点在于计算前的数据排序、计算时替换相同支付金的组合和计算后最优组合的选择.***[项目地址](https://github.com/Thewhitelight/Tinder/blob/master/app/src/test/java/cn/libery/tinder/BestOfferTest.java)***可以运行测试用例.