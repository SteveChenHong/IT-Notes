HTML:
```
<div id="wrapper">
  <div class="child child1">
    111
  </div>
  <div class="child child2">
    222
  </div>
</div>

```

CSS:
```
#wrapper {
  margin-left: 200px;
  width: 200px;
  height: 200px;
  background: lightblue;
  display: flex;
}

.child {
  height: 100%;
  display: inline-block;
}

.child1 {
  /*調整此值可縮放選單*/
  margin-left: -10px;
  width:50px;
  background-color: lightgreen;
}

.child2 {
  width: 100%;
  background-color: lightpink;
}

```
