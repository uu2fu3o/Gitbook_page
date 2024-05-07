### setup语法糖

```vue
<script lang="ts" setup>
	let a = 666
</script>	
```

等效于

```
setup(){
	let a = 666 
	return {a}
}
```

语法糖自动return