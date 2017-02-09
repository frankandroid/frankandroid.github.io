---
title: Recycleview Adapter中出现的一个小bug
date: 2017-02-09 17:47:18
updated: 2017-02-09 17:47:18
categories: [遇到过的小bug]
tags: [遇到过的小bug]
---

# RecycleViewAdapter 使用中遇到的一个小bug

1.先上代码
```java

public class RamdomNumberDialogAdapter extends RecyclerView.Adapter<RamdomNumberDialogAdapter
        .RedViewHolder> {

    private List<Integer> numbers = new ArrayList<>();

    public RamdomNumberDialogAdapter(List<Integer> numbers) {
        this.numbers = numbers;
    }

    @Override
    public RamdomNumberDialogAdapter.RedViewHolder onCreateViewHolder(ViewGroup parent, int
            viewType) {
        return new RedViewHolder(LayoutInflater.from(parent.getContext()).inflate(R.layout
                .adapter_eleven_choice_five_dialog_not_choice_sphere_item, parent, false));
    }

    @Override
    public void onBindViewHolder(RedViewHolder holder, int position) {

        holder.mTextView.setText(numbers.get(position)+"");

        if (position == 6) {
            holder.mTextView.setBackgroundResource(R.drawable.shape_double_item_bg_blue);
        }/*else{
            holder.mTextView.setBackgroundResource(R.drawable.shape_double_item_bg_red);
        }*/
    }

    @Override
    public int getItemCount() {
        return 7;
    }

    public class RedViewHolder extends RecyclerView.ViewHolder {

        TextView mTextView;

        public RedViewHolder(View itemView) {
            super(itemView);

            mTextView = (TextView) itemView.findViewById(R.id.sphere_dialog_text);
        }
    }
}
```

上述代码导致的问题就是每点击一次，蓝色球就会增加一个，其实是复用导致的问题。

<img src="(https://i.niupic.com/images/2017/02/09/hR5uwu.png" />
<img src="https://i.niupic.com/images/2017/02/09/3f4DsO.png" />
<img src="https://i.niupic.com/images/2017/02/09/8YtnJA.png" />

>主要就是holder复用导致的问题，之前也遇到过，现在整理一下。把注释的代码释放就好了

```java
@Override
    public void onBindViewHolder(RedViewHolder holder, int position) {

        holder.mTextView.setText(numbers.get(position)+"");

        if (position == 6) {
            holder.mTextView.setBackgroundResource(R.drawable.shape_double_item_bg_blue);
        }else{
            holder.mTextView.setBackgroundResource(R.drawable.shape_double_item_bg_red);
        }
    }

```
