---
title: 排序
published: 2026-02-25
description: 排序是最基础也是最经典的算法了，其中蕴涵了很多精妙的思想，值得时常复习
tags: [Sort]
category: Coding
draft: true
---
排序是最基础也是最经典的算法了，其中蕴涵了很多精妙的思想，值得时常复习

```java
package com.coding;

import java.util.Arrays;
import java.util.Objects;

public class Sort {

    public static void main(String[] args) {
        int times = 10000;
        int maxValue = 100;
        int maxLength = 100;
        for (int i = 0; i < times; ++i) {
            int[] nums = generateNums(maxLength, maxValue);
            int[] nums1 = Arrays.copyOf(nums, maxLength);
            int[] nums2 = Arrays.copyOf(nums, maxLength);
            Arrays.sort(nums1);
            heapSort(nums2);
            boolean equals = Arrays.equals(nums1, nums2);
            if (!equals) {
                System.err.println("排序出错了");
                break;
            }
        }
    }

    public static int[] generateNums(int maxLength, int maxValue) {
        int length = (int) (Math.random() * maxLength);
        int[] nums = new int[length];
        for (int i = 0; i < length; i++) {
            nums[i] = (int) (Math.random() * maxValue);
        }
        return nums;
    }

    /**
     * 冒泡排序
     * 稳定性：稳定
     * 时间复杂度：O(n^2)
     * 空间复杂度：O(1)
     * @param nums
     */
    public static void bubbleSort(int[] nums) {
        if (Objects.isNull(nums) || nums.length <= 1) {
            return;
        }

        for (int i = 0; i < nums.length - 1; ++i) {
            boolean sorted = true;
            for (int j = 0; j < nums.length - i - 1; ++j) {
                if (nums[j] > nums[j + 1]) {
                    swap(nums, j, j + 1);
                    sorted = false;
                }
            }
            if (sorted) {
                break;
            }
        }
    }

    /**
     * 插入排序
     * 稳定性：稳定
     * 时间复杂度：O(n^2)
     * 空间复杂度：O(1)
     * @param nums
     */
    public static void insertionSort(int[] nums) {
        if (Objects.isNull(nums) || nums.length <= 1) {
            return;
        }
        for (int i = 1; i < nums.length; ++i) {
            for (int j = i; j > 0; --j) {
                if (nums[j] < nums[j - 1]) {
                    swap(nums, j, j - 1);
                } else {
                    break;
                }
            }
        }
    }

    /**
     * 选择排序
     * 稳定性：不稳定
     * 时间复杂度：O(n^2)
     * 空间复杂度：O(1)
     * @param nums
     */
    public static void selectionSort(int[] nums) {
        if (Objects.isNull(nums) || nums.length <= 1) {
            return;
        }
        for (int i = 0; i < nums.length; ++i) {
            int minPos = i;
            int j = i;
            while (j < nums.length) {
                if (nums[j] < nums[minPos]) {
                    minPos = j;
                }
                ++j;
            }
            swap(nums, i, minPos);
        }
    }

    /**
     * 归并排序
     * 稳定性：稳定
     * 时间复杂度：O(n * log_n)
     * 空间复杂度：这个实现方式是O(n * log_n)，但实际可以优化至O(n)
     * @param nums
     */
    public static void mergeSort(int[] nums) {
        if (Objects.isNull(nums) || nums.length <= 1) {
            return;
        }
        mergeSort(nums, 0, nums.length);
    }

    public static void mergeSort(int[] nums, int left, int right) {
        if (right - left <= 1) {
            return;
        }
        int mid = left + (right - left) / 2;
        mergeSort(nums, left, mid);
        mergeSort(nums, mid, right);
        merge(nums, left, mid, right);
    }

    public static void merge(int[] nums, int left, int mid, int right) {
        int[] temp = new int[right - left];
        int index1 = left;
        int index2 = mid;
        int i = 0;
        while (index1 < mid && index2 < right) {
            temp[i++] = nums[index1] <= nums[index2] ? nums[index1++] : nums[index2++];
        }
        while (index1 < mid) {
            temp[i++] = nums[index1++];
        }
        while (index2 < right) {
            temp[i++] = nums[index2++];
        }
        System.arraycopy(temp, 0, nums, left, temp.length);
    }

    /**
     * 快速排序
     * 稳定性：不稳定
     * 时间复杂度：O(n * log_n)
     * 空间复杂度：O(n)
     * @param nums
     */
    public static void quickSort(int[] nums) {
        if (Objects.isNull(nums) || nums.length <= 1) {
            return;
        }
        quickSort(nums, 0, nums.length);
    }

    public static void quickSort(int[] nums, int left, int right) {
        if (right - left <= 1) {
            return;
        }
        int x = nums[left + (int) (Math.random() * (right - left))];
        int[] startAndEnd = partition(nums, x, left, right);
        quickSort(nums, left, startAndEnd[0]);
        quickSort(nums, startAndEnd[1], right);
    }

    public static int[] partition(int[] nums, int x, int left, int right) {
        int leftBound = left;
        int rightBound = right - 1;
        int index = left;
        while (index <= rightBound) {
            if (nums[index] < x) {
                swap(nums, index++, leftBound++);
            } else if (nums[index] > x) {
                swap(nums, index, rightBound--);
            } else {
                ++index;
            }
        }
        return new int[]{leftBound, rightBound + 1};
    }

    /**
     * 堆排序：
     * 稳定性：不稳定
     * 时间复杂度：O(n * log_n)
     * 空间复杂度：O(1)
     * @param nums
     */
    public static void heapSort(int[] nums) {
        for (int i = 0; i < nums.length; ++i) {
            heapInsert(nums, i);
        }
        for (int i = nums.length; i > 0; --i) {
            swap(nums, 0, i - 1);
            heapify(nums, 0, i - 1);
        }
    }

    public static void heapInsert(int[] nums, int i) {
        while (nums[i] > nums[(i - 1) / 2]) {
            swap(nums, i , (i - 1) / 2);
            i = (i - 1) / 2;
        }
    }

    public static void heapify(int[] nums, int i, int size) {
        int left = i * 2 + 1;
        while (left < size) {
            int right = left + 1;
            int best = right < size && nums[left] < nums[right] ? right : left;
            if (nums[i] >= nums[best]) {
                break;
            }
            swap(nums, i, best);
            i = best;
            left = i * 2 + 1;
        }
    }

    public static void swap(int[] nums, int i, int j) {
        int temp = nums[i];
        nums[i] = nums[j];
        nums[j] = temp;
    }
}

```