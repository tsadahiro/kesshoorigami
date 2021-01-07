<template>
<div>
  <svg width="800" height="600" :viewBox="view"
       @mousedown="mouseDown($event)"
       @mousemove="mouseMove($event)"
       @mouseup="mouseUp($event)"
       style="border:solid 1px #000000"
       >
    <face v-for="(f,i) in F" :points="points(f)" :faceup="faceup && faceup[i]"></face>
    <edge v-if="subdivided && !folded" v-for="(e,i) in E" :edge="edge(i)"></edge>
  </svg><br/>
  <v-btn @click="makeTriangles(2)">triangle</v-btn>
  <v-btn v-if="!subdivided" @click="subdivide"> subdivide </v-btn>
  <v-btn v-if="subdivided && !folded" @click="foldAll()"> fold </v-btn>
  <v-btn v-if="subdivided && folded" @click="folded=false"> unfold </v-btn>
  <v-slider :label="'縮小:'+contract" v-model="contract" min="10" max="100"></v-slider>
  <v-slider :label="'回転:'+angle" v-model="angle" min="-90" max="90"></v-slider>
  <v-slider label="スケール" v-model="exp" min="50" max="200"></v-slider>
</div>
</template>

<script>
import face from "~/components/face.vue"
import edge from "~/components/edge.vue"
import vertex from "~/components/vertex.vue"

export default {
    data(){
	return {
	    subdivided: false,
            V:null,//頂点データ (x,y)
            E:null,//辺データ [from ,to]
            F:null,//面データ 辺のindexの集合
	    mountain:null, //山折り:true 谷折り:false
	    directions:null,//各面が順: true , 逆: false
	    faceup:null,//表向き: true, 裏向き: false,
	    cover:null,//上に来る面のindex
	    lower:null,//下に来る面のindex
	    oV: null,
	    oE: null,
	    oF: null,
	    exp: 100,
	    contract: 100,
	    prevContract: 100,
	    angle: 0,
	    trajectory:null,
	    foldedV:null,
	    folded:false,
	    Tree:null,
	    isMouseDown: false,
	    view:"0 0 800 600",
	    viewX: 0,
	    viewY: 0,
	    prevX: 0,
	    prevY: 0,
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
	    if (this.folded){
		this.foldAll();
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
	    if (this.folded){
		this.foldAll();
	    }
	}
    },
    
    computed:{
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
    },
    
    methods:{
	mouseDown(e){
	    console.log(e.offsetX + "," + e.offsetY);
	    this.isMouseDown = true;
	    this.prevX = e.offsetX;//動く前
	    this.prevY = e.offsetY; 
	},
	mouseUp(e){
	    console.log(e.offsetX + "," + e.offsetY);
	    this.isMouseDown = false;//マウスの押し下げが終わった
	},
	mouseMove(e){
	    if(this.isMouseDown){
		let deltaX = e.offsetX - this.prevX;//e.offsetは動いた後の場所
		let deltaY = e.offsetY - this.prevY;
		this.viewX += deltaX;
		this.viewY += deltaY;
		this.view = (-this.viewX) + " " + (-this.viewY) + " 800 800";
		this.prevX = e.offsetX;
		this.prevY = e.offsetY;
	    }
	},
	x1(e){
	    if(this.folded){
		return this.foldedV[e[0]].x*this.exp+this.exp;
	    }
	    else{
		return this.V[e[0]].x*this.exp+this.exp;
	    }
	},
	y1(e){
	    if(this.folded){
		return this.foldedV[e[0]].y*this.exp+this.exp;
	    }
	    else{
		return this.V[e[0]].y*this.exp+this.exp;
	    }
	},
	x2(e){
	    if(this.folded){
		return this.foldedV[e[1]].x*this.exp+this.exp;
	    }
	    else{
		return this.V[e[1]].x*this.exp+this.exp;
	    }
	},
	y2(e){
	    if (this.folded){
		return this.foldedV[e[1]].y*this.exp+this.exp;
	    }
	    else{
		return this.V[e[1]].y*this.exp+this.exp;
	    }
	},
	
	edge(i){
	    const from  = {x:this.V[this.E[i][0]].x, y: this.V[this.E[i][0]].y};
	    const to  = {x:this.V[this.E[i][1]].x, y: this.V[this.E[i][1]].y};
	    let color = null;
	    if (this.mountain && this.mountain[i]){
		color = "blue";
	    }
	    else{
		color = "red";
	    }
	    //console.log({from:from, to:to, color:color});
	    return {from:from, to:to, color:color};
	},
	
	dist(p,q){
	    return Math.sqrt((p.x-q.x)*(p.x-q.x) + (p.y-q.y)*(p.y-q.y));
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
	    let mountain = []
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
		    mountain.push(true);
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
	    for (let eidx in this.E){//オリジナルのセル分割の各辺について面を作る
		// オリジナルの各辺はそれが２面に共有されたものであったら
		// 上の操作で２つに分裂しており、
		// fidx,sidxにはそれぞれの新しいEにおけるindexが入る。
		let fidx = ancE.indexOf(1*eidx)
		let sidx = ancE.lastIndexOf(1*eidx)

		
		if (fidx!=sidx){
		    let fFaceIndex = F.findIndex((f)=>(f.indexOf(fidx)>=0))
		    let sFaceIndex = F.findIndex((f)=>(f.indexOf(sidx)>=0))
		    //console.log(fFaceIndex,sFaceIndex)
		    console.log(fFaceIndex, ":", this.directions[fFaceIndex], sFaceIndex, ":", this.directions[sFaceIndex]);
		    let vec1 = {x:this.V[this.E[eidx][1]].x - this.V[this.E[eidx][0]].x,
				y:this.V[this.E[eidx][1]].y - this.V[this.E[eidx][0]].y}
		    let mcenter = this.mCenter(this.F[fFaceIndex]);
		    let vec2 = {x:mcenter.x - this.V[this.E[eidx][0]].x,
				y:mcenter.y - this.V[this.E[eidx][0]].y}
		    let leftside = vec1.x*vec2.y-vec1.y*vec2.x;
		    let edge0 = [E[fidx][0], E[sidx][0]]
		    let edge1 = [E[fidx][1], E[sidx][1]]
		    console.log(vec1,vec2);
		    console.log(leftside);
		    //console.log("newedge", ancV[E[fidx][0]], ancV[E[fidx][1]])
		    E.push(edge0)
		    mountain.push(this.directions[fFaceIndex]*leftside > 0 ? true : false);
		    E.push(edge1)
		    mountain.push(this.directions[fFaceIndex]*leftside > 0 ? false : true);
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
	    this.mountain = mountain;

	    //console.log(this.Tree)
	    this.Tree = this.makeTree();
	    this.subdivided = true
	    this.makeCoverRelation();
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

	commonEdge: function(i,j){
	    if (i==j){
		return -1
	    }
	    for (let e of this.F[i]){
		let cidx = this.F[j].indexOf(e)
		if (cidx > -1){
		    return e;
		}
	    }
	    return -1;
	},
	
	ref(p,e){ // p ={x:, y:}, e is the index of E
	    console.log(p,e);
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
	    this.foldedV = [];
	    for (let i in this.V){
		this.foldedV.push(null);
	    }
	    for (let f in this.F){
		//console.log(this.foldToRoot(f));
		let foldedImg = this.foldToRoot(f);
		console.log(foldedImg)
		for (let i in foldedImg.index){
		    this.foldedV[foldedImg.index[i]] = foldedImg.points[i];
		}
		console.log(this.V.length, this.foldedV.length);
	    }
	    this.folded=true;
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
		console.log("face:",findex, this.Tree[findex])
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
		//for (let q=p+1; q < coefs.length; q++){
		for (let q in coefs){
		    if (p >= q) continue;
		    if (Math.abs(coefs[p].a-coefs[q].a)==1 && coefs[p].b == coefs[q].b){
			E.push([1*p,1*q]);
		    }
		    if (Math.abs(coefs[p].b-coefs[q].b)==1 && coefs[p].a == coefs[q].a){
			E.push([1*p,1*q]);
		    }
		    if (coefs[p].b-coefs[q].b == 1 && coefs[q].a - coefs[p].a == 1){
			E.push([1*p,1*q]);
		    }
		    if (coefs[p].b-coefs[q].b == -1 && coefs[q].a - coefs[p].a == -1){
			E.push([1*p,1*q]);
		    }
		}
	    }
	    console.log(E)
	    const F = [];
	    for (let a in E){
		for (let b in E){
		    if (a >= b) continue
		    if (E[a][1] == E[b][0] ){
			let min = Math.min(E[a][0],E[b][1])
			let max = Math.max(E[a][0],E[b][1])
			for (let c in E){
			    if ((E[c][0]==min && E[c][1]==max) || (E[c][1]==min && E[c][0]==max)){
				F.push([a,b,c]);
			    }
			}
		    }
		    if (E[a][0] == E[b][1] ){
			let min = Math.min(E[a][1],E[b][0])
			let max = Math.max(E[a][1],E[b][0])
			for (let c in E){
			    if ((E[c][0]==min && E[c][1]==max) || (E[c][1]==min && E[c][0]==max)){
				F.push([a,b,c]);
			    }
			}
		    }
		}
	    }
	    console.log(F)
	    this.V = V;
	    this.E = E;
	    this.F = F;
	},

	makeTree(){
	    let tree =[]
	    let inTree = []
	    let centers = []
	    let sum = {x:0,y:0}

	    for (let f in this.F){
		inTree.push(false)
		tree.push(0)
		centers.push(this.mCenter(f));
		sum.x += this.mCenter(f).x;
		sum.y += this.mCenter(f).y;
	    }
	    const totalCenter = {x:(sum.x/(this.F.length)), y:(sum.y/(this.F.length))}
	    console.log("total center:",totalCenter)
	    let rindex = 0;
	    console.log(centers[rindex]);
	    let minDist = this.dist(centers[rindex],totalCenter);
	    console.log(minDist);
	    for (let c in centers){
		if (this.dist(centers[c],totalCenter)<minDist){
		    rindex = 1*c;
		}
	    }
	    inTree[rindex]=true
	    let boundary = [rindex]
	    tree[rindex]=rindex;
	    this.faceup = [];
	    let faceup = true;
	    this.faceup[rindex] = faceup;

	    let newBoundary = []
	    while(boundary.length > 0){
		faceup = !faceup;
		for (let b of boundary){
		    for (let n of this.adjList[b]){
			if (!inTree[n]){
			    tree[n] = b
			    this.faceup[n] = faceup;
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

	makeCoverRelation(){
	    let cover = [];
	    let lower = [];
	    for (let i in this.F){
		let C = [];
		let L = [];
		for (let j in this.F){
		    let common = this.commonEdge(i,j)
		    if (common >= 0){
			if ((this.faceup[i] && this.mountain[common]) || (!this.faceup[i] && !this.mountain[common])){
			    C.push(j);
			}
			else{
			    L.push(j);
			}
		    }
		}
		cover.push(C);
		lower.push(L);
	    }
	    this.cover=cover;
	    this.lower=lower;
	    console.log("cover",cover)
	    console.log("lower",lower);
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
	this.makeSquares(5,5);
    },


    components: {
	face,
	edge,
	vertex
    }
}
</script>
