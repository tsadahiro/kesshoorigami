<template>
<div>
  <v-btn @click="makeTriangles(3)">triangle</v-btn>
  <v-btn v-if="!subdivided" @click="subdivide"> subdivide </v-btn>
  <v-btn v-if="subdivided" @click="foldAll()"> fold </v-btn>
  <svg width="800" height="800">
    <polygon v-for="f in F" :points="points(f)" fill="green" fill-opacity="0.3"></polygon>
    <line v-for="e in E" :x1="x1(e)" :y1="y1(e)" :x2="x2(e)" :y2="y2(e)" stroke="blue"> </line>
    <!--
    <circle v-for="v in V" :cx="v.x*exp+exp" :cy="v.y*exp+exp" r="2"></circle>
    <circle v-if="M.length>0" v-for="m in M" :cx="m.x*exp+exp" :cy="m.y*exp+exp" r="2" stroke="red"></circle>
	 <line v-for="(to,from) in Tree" :x1="M[from].x*exp+exp" :y1="M[from].y*exp+exp"
	       :x2="M[to].x*exp+exp" :y2="M[to].y*exp+exp" stroke="red"></line> 
	 -->
  </svg>
  <v-slider label="縮小" v-model="contract" min="10" max="100"></v-slider>
  <v-slider label="回転" v-model="angle" min="-45" max="45"></v-slider>
  <v-slider label="スケール" v-model="exp" min="50" max="200"></v-slider>
</div>
</template>

<script>
export default {
    data(){
	return {
	    subdivided: false,
            V:null,
            E:null,
            F:null,
	    directions:null,
	    oV: null,
	    oE: null,
	    oF: null,
	    exp: 100,
	    contract: 100,
	    prevContract: 100,
	    angle: 0,
	    trajectory:null,
	}
    },
    watch:{
	contract:function(){
	    if (!this.subdivided) return;
	    for (let i in this.oF){
		let vset = []
		for (let e of this.oF[i]){
		    for (let v of this.E[e]){
			if (vset.find((e)=>{e==v})===undefined){
			    vset.push(v)
			}
		    }
		}
		let sum = {x:0,y:0}
		for (let v of vset){
		    sum.x += this.V[v].x
		    sum.y += this.V[v].y
		}
		let m={x:sum.x/vset.length, y:sum.y/vset.length}
		let c=this.contract/100;
		for (let v of vset){
		    let vertex = {x: c*this.oV[v].x + (1-c)*m.x,
				  y: c*this.oV[v].y + (1-c)*m.y}
		    let theta = this.directions[i]*Math.PI*this.angle/180
		    vertex = {x: (vertex.x-m.x)*Math.cos(theta) - (vertex.y-m.y)*Math.sin(theta) + m.x,
			      y: (vertex.x-m.x)*Math.sin(theta) + (vertex.y-m.y)*Math.cos(theta) + m.y}
		    this.V.splice(v,1,vertex)
		}
	    }
	},
	angle(){
	    if (!this.subdivided) return;
	    //for (let f of this.oF){
	    for (let i in this.oF){
		let vset = []
		for (let e of this.oF[i]){
		    for (let v of this.E[e]){
			if (vset.find((e)=>{e==v})===undefined){
			    vset.push(v)
			}
		    }
		}
		let sum = {x:0,y:0}
		for (let v of vset){
		    sum.x += this.V[v].x
		    sum.y += this.V[v].y
		}
		let m={x:sum.x/vset.length, y:sum.y/vset.length}
		let c=this.contract/100;
		for (let v of vset){
		    let vertex = {x: c*this.oV[v].x + (1-c)*m.x,
				  y: c*this.oV[v].y + (1-c)*m.y}
		    let theta = this.directions[i]*Math.PI*this.angle/180
		    vertex = {x: (vertex.x-m.x)*Math.cos(theta) - (vertex.y-m.y)*Math.sin(theta) + m.x,
			      y: (vertex.x-m.x)*Math.sin(theta) + (vertex.y-m.y)*Math.cos(theta) + m.y}
		    this.V.splice(v,1,vertex)
		}
	    }
	}
    },

    computed:{
	M(){
	    const C = []
	    for (let f of this.F){
		C.push(this.mCenter(f))
	    }
	    console.log(C)
	    return C;
	},
	adjList(){ // 面と面が辺を共有するとき、隣接すると考える。
	    let A = [];
	    for (let i in this.F){
		let R = [];
		for (let j in this.F){
		    if (this.isAdjacent(i,j)){
			R.push(j);
		    }
		}
		A.push(R);
	    }
	    //console.log(A)
	    return A;
	},
	Tree(){
	    let tree =[]
	    let inTree = []
	    let boundary = [0]

	    for (let f in this.F){
		inTree.push(false)
		tree.push(0)
	    }
	    inTree[0]=true

	    let newBoundary = []
	    while(boundary.length > 0){
		for (let b of boundary){
		    for (let n of this.adjList[b]){
			if (!inTree[n]){
			    tree[n] = b
			    newBoundary.push(1*n)
			    inTree[n]=true
			}
		    }
		}
		boundary = newBoundary.slice(0,newBoundary.length);
		newBoundary = []
	    }
	    console.log(tree)
	    return tree
	},
    },
    
    methods:{
	x1(e){
            return this.V[e[0]].x*this.exp+this.exp;
	},
	y1(e){
            return this.V[e[0]].y*this.exp+this.exp;
	},
	x2(e){
            return this.V[e[1]].x*this.exp+this.exp;
	},
	y2(e){
            return this.V[e[1]].y*this.exp+this.exp;
	},
	points(f){
	    let seq = []
	    let pts = ""
	    seq.push(this.E[f[0]][0])
	    seq.push(this.E[f[0]][1])
	    let used = []
	    for (let e in this.E){
		used.push(false)
	    }
	    used[f[0]] = true;
	    pts = pts + " " + this.x1(this.E[f[0]]) + "," + this.y1(this.E[f[0]])
	    
	    while (seq.length <= f.length){
		for (let eidx of f){
		    if (used[eidx]) continue;

		    let last = seq[seq.length-1]
		    if (this.E[eidx][0]==last && this.E[eidx][1]!=last){
			used[eidx] = true;
			seq.push(this.E[eidx][1])
			pts = pts + " " + this.x1(this.E[eidx]) + "," + this.y1(this.E[eidx])
			break;
		    }
		    if (this.E[eidx][1]==last && this.E[eidx][0]!=last){
			used[eidx] = true;
			seq.push(this.E[eidx][0])
			pts = pts + " " + this.x2(this.E[eidx]) + "," + this.y2(this.E[eidx])
			break;
		    }
		}
	    }
	    return pts;
	},

	directFaces(){
	    let directions =[]
	    let directed = []
	    let boundary = [0]

	    for (let f in this.F){
		directed.push(false)
		directions.push(1)
	    }
	    directed[0]=true

	    let newBoundary = []
	    while(boundary.length > 0){
		for (let b of boundary){
		    for (let n of this.adjList[b]){
			if (!directed[n]){
			    directions[n] = -directions[b]
			    newBoundary.push(1*n)
			    directed[n]=true
			}
		    }
		}
		boundary = newBoundary.slice(0,newBoundary.length);
		newBoundary = []
	    }
	    this.directions = directions;
	    console.log("directed")
	    console.log(directions)
	},
	
	subdivide(){
	  let F = []
	  let E = []
	  let V = []
	  let ancF = []
	  let ancE = []
	  let ancV = []
	  let vFaces = []
	  for (let v in this.V){
	      vFaces.push([])
	  }
	  this.directFaces()
	  for (let f of this.F){
	      let findices = []
	      let count = 0
	      for (let e of f){
		  let edge = []
		  for (let v of this.E[e]){
		      if (findices[v] === undefined){
			  findices[v]=V.length
			  V.push(this.V[v])
			  ancV.push(v)
			  count++
		      }
		      edge.push(findices[v])
		  }
		  E.push(edge)
		  ancE.push(1*e)
	      }
	      let face = []
	      for (let i=0; i<f.length; i++){
		  face.push(E.length-f.length+i)
	      }
	      F.push(face)
	      ancF.push(f)
	  }
	  //console.log(F.length, E.length, V.length)
	  this.oV = V.slice(0,V.length);
	  this.oE = E.slice(0,E.length);
	  this.oF = F.slice(0,F.length);
	  //console.log(ancE)
	  for (let eidx in this.E){//各辺について面を作る
	      let fidx = ancE.indexOf(1*eidx)
	      let sidx = ancE.lastIndexOf(1*eidx)
	      
	      if (fidx!=sidx){
		  let edge0 = [E[fidx][0], E[sidx][0]]
		  let edge1 = [E[fidx][1], E[sidx][1]]
		  //console.log("newedge", ancV[E[fidx][0]], ancV[E[fidx][1]])
		  E.push(edge0)
		  E.push(edge1)
		  vFaces[ancV[E[fidx][0]]].push(E.length-2)
		  vFaces[ancV[E[fidx][1]]].push(E.length-1)
		  F.push([fidx, sidx, E.length-2, E.length-1])
	      }
	  }

	  //console.log(vFaces.filter((x)=>x.length>2))
	  //console.log(F)
	  this.V = V;
	  this.E = E;
	  this.F = F.concat(vFaces.filter((x)=>x.length>2)); // ここ変です。

	  console.log(this.Tree)
	  this.subdivided = true
      },

      mCenter(f){
	  const V = []
	  let sum = {x:0,y:0}
	  for (let e of f){
	      for (let v of this.E[e]){
		  if (V.indexOf(v)<0){
		      V.push(v)
		      sum.x += this.V[v].x
		      sum.y += this.V[v].y
		      //console.log(v,this.V[v].x, this.V[v].y)
		  }
	      }
	  }
	  //console.log(V.length)
	  return {x: sum.x/V.length, y: sum.y/V.length}
      },
      
	isAdjacent: function(i,j){
	    if (i==j){
		return false
	    }
	    for (let e of this.F[i]){
		if (this.F[j].indexOf(e) > -1){
		    return true;
		}
	    }
	    return false;
	},

	ref(p,e){ // p ={x:, y:}, e is the index of E
	    const ax = this.V[this.E[e][0]].x;
	    const ay = this.V[this.E[e][0]].y;
	    const bx = this.V[this.E[e][1]].x;
	    const by = this.V[this.E[e][1]].y;
	    const Tp = {x:p.x,y:p.y}

	    const normSquared = (ax-bx)*(ax-bx)+(ay-by)*(ay-by)
	    const dotProd = (p.x-ax)*(-by+ay)+(p.y-ay)*(bx-ax)

	    Tp.x -= 2*dotProd/normSquared*(-by+ay)
	    Tp.y -= 2*dotProd/normSquared*(bx-ax)

	    return Tp
	},
	
	foldAll(){
	    let V = [];
	    for (let i in this.V){
		V.push(null);
	    }
	    for (let f in this.F){
		//console.log(this.foldToRoot(f));
		let foldedImg = this.foldToRoot(f);
		console.log(foldedImg)
		for (let i in foldedImg.index){
		    V[foldedImg.index[i]] = foldedImg.points[i]
		}
	    }
	    this.V = V
	},

	foldToRoot(findex){
	    if (!this.subdivided) return;
	    
	    let isRoot = function(v,tree){
		if (tree[v] == v){
		    return true;
		}
		else{
		    return false;
		}
	    };
	    
	    let vset = []
	    for (let e of this.F[findex]){
		if (vset.indexOf(this.E[e][0])<0){
		    vset.push(this.E[e][0])
		}
		if (vset.indexOf(this.E[e][1])<0){
		    vset.push(this.E[e][1])
		}
	    }
	    let points = []
	    for (let v of vset){
		points.push(this.V[v])
	    }
	    while (!isRoot(findex,this.Tree)){
		//console.log(points)
		let to = this.Tree[findex]
		let edge = null
		for (let e of this.F[findex]){
		    if (this.F[to].indexOf(e)>=0){
			edge = e
			break
		    }
		}
		let newPoints = []
		for (let p of points){
		    newPoints.push(this.ref(p,edge))
		    //console.log(this.V[v], " |--> ", this.ref(this.V[v],edge))
		}
		points = newPoints;
		findex = this.Tree[findex]
	    }
	    console.log("folded:",points)
	    return {index:vset,points:points}
	},

	    
	makeTriangles(n){
	    const V=[]
	    const coefs = []
	    for (let a=-n; a<=n; a++){
		for (let b=-n; b<=n; b++){
		    if (Math.abs(a+b)<=n){
			V.push({x: a+b/2, y: b*Math.sqrt(3)/2})
			coefs.push({a:a, b:b})
		    }
		}
	    }
	    const E=[]
	    for (let p in coefs){
		for (let q in coefs){
		    if (Math.abs(coefs[p].a-coefs[q].a)==1 && coefs[p].b == coefs[q].b){
			E.push([p,q]);
		    }
		}
	    }
	},
	
      makeSquares(m,n){

	  let mdist = function(p,q){
	      return Math.abs(p.x-q.x) + Math.abs(p.y-q.y)
	  };
	  
	  let V = [];
	  for (let i = 0; i < n; i++){
	      for(let j = 0; j < m; j++){
		  V.push({x:i, y:j})
	      }
	  }
	  let E = [];
	  for (let p in V){
	      for (let q in V){
		  if (mdist(V[p],V[q]) == 1 && (V[p].x <= V[q].x) && (V[p].y <= V[q].y)){
		      E.push([p,q])
		  }
	      }
	  }
	  
	  let F = [];

	  for (let i=0; i< n-1; i++){
	      for (let j=0; j<m-1; j++){
		  let f = []
		  let center = {x:(i+0.5), y:(j+0.5)}
		  for (let e in E){
		      if (mdist(center, V[E[e][0]]) <= 1 &&  mdist(center, V[E[e][1]]) <= 1){
			  f.push(e)
			  //console.log(center)
			  //console.log(V[E[e][0]])
			  //console.log(V[E[e][1]])
		      }
		  }
		  F.push(f)
	      }
	  }
	  this.V = V;
	  this.E = E;
	  this.F = F;

	  console.log(V);
	  console.log(E);
	  console.log(F);
	  console.log(V.length);
	  console.log(E.length);
	  console.log(F.length);
      },

  },

    created(){
	this.makeSquares(6,6);
	console.log("test");
    },


    components: {
  }
}
</script>
