<template>
<g>
  <line :x1="from.x" :y1="from.y" :x2="to.x" :y2="to.y"
	:stroke="edge.color"></line>
</g>
</template>

<script>
export default{
    props:['edge'],

    computed:{
	exp(){
	    return this.$store.state.exp;
	},
	
	from(){
	    return {x:(this.edge.from.x+1)*this.exp,
		    y:(this.edge.from.y+1)*this.exp}
	},
	
	to(){
	    return {x:(this.edge.to.x+1)*this.exp,
		    y:(this.edge.to.y+1)*this.exp}
	},
	
    }
}
</script>
