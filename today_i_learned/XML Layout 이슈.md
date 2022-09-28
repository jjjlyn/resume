```xml
<ViewPager>
  <RecyclerView> 
  --> (when itemView includes RecyclerView) 
   ex) <ConstraintLayout>
          <TextView>
          <RecyclerView>
```

반드시 RecyclerView의 아이템에 Nested RecyclerView가 있으면 `layout_height`을 `wrap_content`로 설정해야 스크롤이 작동한다.
