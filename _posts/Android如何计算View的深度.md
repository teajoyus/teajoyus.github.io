今天在QQ群上，看到大家谈到了一个面试题，就是如何求View树的深度。在我们项目中基本上比较少需要到这个计算，所以可能一下子会蒙圈了。
我们知道，Android的视图是一颗树的形式，那么即使关于Android的View树方面很多计算，便可以利用树的原理来计算。
谈到树，我们在书本上最常看到的就是二叉树，项目上也有很多关于树的影子，比如有个栏目接口，栏目内容是一层套一层的，那么也是一种树的表现。

如果我们想获得一颗二叉树的深度，简单的做法可以利用递归的思想来做：
```
int maxDeep(TreeNode *root)
{
	if(root == NULL)
		return 0;
	int max1 = maxDeep(root->left) + 1;
	int max2 = maxDeep(root->right) + 1;
	return (max1 > max2) ? max1 : max2;
}
```

如何理解上面几句代码？

首先root如果是NULL的话返回0，代表如果遍历到叶子结点已经是空了，那么这一层根本不算1层，所以返回0，
然后计算int max1 = maxDeep(root->left) + 1; 直接取得左子树的深度，+1是因为root不为空，也就是当前结点不为空，那么就要累加1层，然后再调用maxDeep去计算左子树的深度

同理	int max2 = maxDeep(root->right) + 1;计算右子树的深度

最后左右子树哪个大就返回哪个。

那么View树也是同样的道理，有人说，View树不是二叉树，不能这么算吧。

这说明没有本质上理解树，只停留在二叉树的代码理解上。

由于上面是二叉树，所以只需要计算左子树右子树，那么如果是三叉树呢？三叉树那就是求左中右三个子树，哪个层级最大便使用哪个。重点在于求各个子树，然后比较出各个子树之间谁是最大的层级的。

依此，View树就是这个道理。

代码如下：
```
 private int maxDeep(View view) {
        //当前的view已经是最底层view了，不能往下累加层数了，返回0，代表view下面只有0层了
        if (!(view instanceof ViewGroup)) {
            return 0;
        }
        ViewGroup vp = (ViewGroup) view;
        //虽然是viewgroup，但是如果并没有任何子view，那么也已经是最底层view了，不能往下累加层数了，返回0，代表view下面只有0层了
        if (vp.getChildCount() == 0) {
            return 0;
        }
        //用来记录最大层数
        int max = 0;
        //广度遍历view
        for (int i = 0; i < vp.getChildCount(); i++) {
            //由于vp拥有子view，所以下面还有一层，因为可以+1，来叠加一层，然后再递归几岁算它的子view的层数
            int deep = maxDeep(vp.getChildAt(i)) + 1;
            //比较哪个大就记录哪个
            if (deep > max) {
                max = deep;
            }
        }
        return max;
    }
```

读者可以自己在XML中建立几个view层级，比如：
```
 <RelativeLayout
        android:id="@+id/rl_1"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content">

        <View
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"></View>

        <RelativeLayout
            android:id="@+id/rl_2"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content">

            <View
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"></View>

            <View
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"></View>

            <RelativeLayout
                android:id="@+id/rl_3"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content">

                <View
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"></View>

                <View
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"></View>

            </RelativeLayout>
        </RelativeLayout>
    </RelativeLayout>
```

比如上面，如果吧rl_1这个当做根View的话。（注意这里根View不算一层）我们自己看的话就能看得出最大深度是3

```
    View view = findViewById(R.id.rl_1);
    //打印出了3
    Log.i("222222222", maxDeep(view) + "");
```












