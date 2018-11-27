title: SOP_控件_AlertDialog
date: 2018-08-11 11:12:05
tags: 

- RockChip
- Android

---

## AlertDialog 使用步骤

```java
            AlertDialog.Builder builder = new AlertDialog.Builder(context);
            builder.setTitle("Warning");
            builder.setMessage("You are forced to be offline. Please try to login again.");
            builder.setCancelable(false); // 设置为不可取消
            builder.setPositiveButton("OK", new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    ActivityCollector.finishAll(); // 销毁所有的活动
                    Intent intent = new Intent(context , LoginActivity.class);
                    context.startActivity(intent); // 重新启动 LoginActivity
                }
            });
            builder.show();
```