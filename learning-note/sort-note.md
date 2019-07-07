```java
package com.liang.tind.www.tindtest.activty;

import java.util.Arrays;

/**
 * created by sherlock
 * <p>
 * date 2019/6/29
 */
public class Test {
    private static int[] arr = new int[]{
            12, 23, 56, 258, 78, 44, 46, 2, 36
    };
    
    //选择排序
    @org.junit.Test
    public void selectSort() throws Exception {
        int minIndex;
        for (int i = 0; i < arr.length - 1; i++) {
            minIndex = i;

            for (int j = i + 1; j < arr.length; j++) {
                if (arr[j] < arr[minIndex]) {
                    minIndex = j;
                }
            }

            if (minIndex != i) {
                int temp = arr[i];
                arr[i] = arr[minIndex];
                arr[minIndex] = temp;
            }
        }

        System.out.println(Arrays.toString(arr));
    }
    
    //冒泡排序
    @org.junit.Test
    public void popSort() throws Exception {
        for (int i = 0; i < arr.length - 1; i++) {
            for (int j = 0; j < arr.length - 1 - i; j++) {
                if (arr[j] > arr[j + 1]) {
                    int temp = arr[j];
                    arr[j] = arr[j + 1];
                    arr[j + 1] = temp;
                }
            }
        }

//        for(int i=0; i<arr.length-1; i++) {
//            for(int j=arr.length-1; j>i; j--) {
//                if(arr[j] < arr[j-1]) {
//                    int temp = arr[j];
//                    arr[j] = arr[j-1];
//                    arr[j-1]  =temp;
//                }
//            }
//        }

        System.out.println(Arrays.toString(arr));
    }

    //插入排序
    @org.junit.Test
    public void insertSort() throws Exception {
        for (int i = 1; i < arr.length; i++) {
            int j = i;
            int temp = arr[i];

            while (j > 0 && temp < arr[j - 1]) {
                // 后移
                arr[j] = arr[j - 1];
                j--;
            }
            //j-- 后插入
            arr[j] = temp;
        }

        System.out.println(Arrays.toString(arr));
    }
    
    //快速排序
    @org.junit.Test
    public void quickSort() throws Exception {
        realQuickSort(arr, 0, arr.length - 1);
        System.out.println(Arrays.toString(arr));
    }

    private void realQuickSort(int[] arr, int low, int high) {
        if (arr == null || arr.length == 0)
            return;
        if (low < high) {
            int index = getIndex(arr, low, high);
            realQuickSort(arr, low, index - 1);
            realQuickSort(arr, index + 1, high);
        }
    }

    private int getIndex(int[] arr, int low, int high) {
        int temp = arr[low];

        while (low < high) {
            while (low < high && arr[high] >= temp) {
                high--;
            }
            arr[low] = arr[high];

            while (low < high && arr[low] <= temp) {
                low++;
            }

            arr[high] = arr[low];
        }

        arr[low] = temp;

        return low;
    }


    //归并排序
    @org.junit.Test
    public void mergeSort() throws Exception {
        realMergeSort(arr, 0, arr.length - 1);
        System.out.println(Arrays.toString(arr));
    }

    private void realMergeSort(int[] arr, int low, int high) {
        if (arr == null || arr.length == 0 || low >= high)
            return;
        int mid = (low + high) / 2;

        realMergeSort(arr, low, mid);
        realMergeSort(arr, mid + 1, high);
        merge(arr,low, mid,high);
    }

    private void merge(int[] arr, int low, int mid, int high) {
        int[] temp = new int[high - low +1];
        int i = low;
        int j = mid+1;
        int k = 0;
        //把左边的元素逐个和右边的元素比较排序
        while (i <= mid && j <= high){
            if (arr[i] >= arr[j]){
                temp[k++] = arr[j++];
            }else {
                temp[k++] = arr[i++];
            }
        }
        //左边剩下的添加进去
        while (i<=mid){
            temp[k++] = arr[i++];
        }

        while (j <= high){
            temp[k++] = arr[j++];
        }

        System.arraycopy(temp, 0, arr, low, temp.length);
    }
}

  @org.junit.Test
    public void testBinarySearch(){
        mergeSort();
        System.out.println("36的索引="+binarySearch(arr,36));
        System.out.println("999的索引="+binarySearch(arr,999));
    }
    public int binarySearch(int[] arr, int searchValue) {

        int low = 0;
        int high = arr.length - 1;
        while (low <= high) {
            int mid = ( high + low) / 2;
            if (arr[mid] == searchValue){
                return mid;
            }else if (arr[mid] > searchValue){
                high = mid - 1;
            }else {
                low = mid + 1;
            }
        }

        return -1;
    }
```