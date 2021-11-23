---
layout:     post
title:      "Polygon Navmesh"
subtitle:   "Recast Detour 07"
date:       2020-08-07 12:04:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - PathFinding
---

得到寻路区域的轮廓后，下一步是生成多边形，主要是将轮廓转为多个凸多边形，实现代码是rcBuildPolyMesh
```c++
m_pmesh = rcAllocPolyMesh();
rcBuildPolyMesh(m_ctx, *m_cset, m_cfg.maxVertsPerPoly, *m_pmesh)
```
下面对代码具体分析。

### 循环处理每个轮廓

##### 切分为三角形

```c++
bool rcBuildPolyMesh(rcContext* ctx, rcContourSet& cset, const int nvp, rcPolyMesh& mesh)
{
	//初始化，省略

	for (int i = 0; i < cset.nconts; ++i)
	{
		rcContour& cont = cset.conts[i];
		// Skip null contours.
		if (cont.nverts < 3)
			continue;

		// Triangulate contour
		for (int j = 0; j < cont.nverts; ++j)
			indices[j] = j;
		int ntris = triangulate(cont.nverts, cont.verts, &indices[0], &tris[0]);
		if (ntris <= 0) {
			ntris = -ntris;
		}
```
- 生成多边形是一个循环，每次处理一个轮廓。处理的第一个步骤是将当前轮廓切分为多个三角形，过程非常简单，检查每个顶点和相邻的两个点能不能组成有效三角形，新三角形必须在多边形内并且没有和其它边相交。每次选出符合条件的最短的边，直至整个多边形都被划分为三角形
```c++
static int triangulate(int n, const int* verts, int* indices, int* tris)
{
	int ntris = 0;
	int* dst = tris;
	
	// The last bit of the index is used to indicate if the vertex can be removed.
	// 检查i,i1,i2是不是有效三角形，要求i-i2线段在多边形内，而且没有和其它非相邻边相交，结果存放在i1
	for (int i = 0; i < n; i++) {
		int i1 = next(i, n);
		int i2 = next(i1, n);
		if (diagonal(i, i2, n, verts, indices))
			indices[i1] |= 0x80000000;
	}
	
	while (n > 3) {
		int minLen = -1;
		int mini = -1;
		for (int i = 0; i < n; i++) {
			int i1 = next(i, n);
			if (indices[i1] & 0x80000000) {  // 上面得到的结果，这个顶点和相邻两个顶点能够构成有效三角形
				const int* p0 = &verts[(indices[i] & 0x0fffffff) * 4];
				const int* p2 = &verts[(indices[next(i1, n)] & 0x0fffffff) * 4];
				int dx = p2[0] - p0[0];
				int dy = p2[2] - p0[2];
				int len = dx*dx + dy*dy;
				if (minLen < 0 || len < minLen) {  // 找到最短的那条边
					minLen = len;
					mini = i;
				}
			}
		}
		
		// ......特殊情况，省略
		
		int i = mini;
		int i1 = next(i, n);
		int i2 = next(i1, n);
		// 添加本次的三角形到结果
		*dst++ = indices[i] & 0x0fffffff;
		*dst++ = indices[i1] & 0x0fffffff;
		*dst++ = indices[i2] & 0x0fffffff;
		ntris++;
		
		// Removes P[i1] by copying P[i+1]...P[n-1] left one index.
		n--;
		// 移除这个顶点，更新indices相关标记。省略......
		// 计算新增的边
		if (diagonal(i, next(i1, n), n, verts, indices))
			indices[i1] |= 0x80000000;
		else
			indices[i1] &= 0x0fffffff;
	}
	
	// Append the remaining triangle.
	*dst++ = indices[0] & 0x0fffffff;
	*dst++ = indices[1] & 0x0fffffff;
	*dst++ = indices[2] & 0x0fffffff;
	ntris++;
	return ntris;
}
```

##### 合并前处理
- 循环处理轮廓的第二步是准备多边形数据，将顶点数据加入到后面需要处理的数据集中
```c++
		// Add and merge vertices.
		for (int j = 0; j < cont.nverts; ++j) {
			const int* v = &cont.verts[j*4];
			indices[j] = addVertex((unsigned short)v[0], (unsigned short)v[1], (unsigned short)v[2],
								   mesh.verts, firstVert, nextVert, mesh.nverts);
			if (v[3] & RC_BORDER_VERTEX) {
				// This vertex should be removed.
				vflags[indices[j]] = 1;
			}
		}

		// Build initial polygons.
		int npolys = 0;
		memset(polys, 0xff, maxVertsPerCont*nvp*sizeof(unsigned short));
		for (int j = 0; j < ntris; ++j) {
			int* t = &tris[j*3];
			if (t[0] != t[1] && t[0] != t[2] && t[1] != t[2]) {
				polys[npolys*nvp+0] = (unsigned short)indices[t[0]];
				polys[npolys*nvp+1] = (unsigned short)indices[t[1]];
				polys[npolys*nvp+2] = (unsigned short)indices[t[2]];
				npolys++;
			}
		}
		if (!npolys)
			continue;
```

##### 合并多边形
- 这是处理一个轮廓的最后一个步骤，只是用三角形会导致数据量太多，寻路会比较慢，所以应该进行一定合并，转化成凸多边形
```c++
		// Merge polygons.
		if (nvp > 3) {
			for(;;) {
				// Find best polygons to merge.
				int bestMergeVal = 0;
				int bestPa = 0, bestPb = 0, bestEa = 0, bestEb = 0;
				for (int j = 0; j < npolys-1; ++j) {
					unsigned short* pj = &polys[j*nvp];
					for (int k = j+1; k < npolys; ++k) {
						unsigned short* pk = &polys[k*nvp];
						int ea, eb;
						int v = getPolyMergeValue(pj, pk, mesh.verts, ea, eb, nvp);
						if (v > bestMergeVal) {
							bestMergeVal = v;
							bestPa = j;
							bestPb = k;
							bestEa = ea;
							bestEb = eb;
						}
					}
				}
				
				if (bestMergeVal > 0) {
					// Found best, merge.
					unsigned short* pa = &polys[bestPa*nvp];
					unsigned short* pb = &polys[bestPb*nvp];
					mergePolyVerts(pa, pb, bestEa, bestEb, tmpPoly, nvp);
					unsigned short* lastPoly = &polys[(npolys-1)*nvp];
					if (pb != lastPoly)
						memcpy(pb, lastPoly, sizeof(unsigned short)*nvp);
					npolys--;
				}
				else
					break;
			}
		}
		
		// Store polygons.
		// 存储结果，省略......
	}
```
- 主要是找出最合适合并的那条边，计算函数为getPolyMergeValue。合并就是简单地删除重复顶点，调整顺序之类的
```c++
static int getPolyMergeValue(unsigned short* pa, unsigned short* pb,
							 const unsigned short* verts, int& ea, int& eb,
							 const int nvp)
{
	const int na = countPolyVerts(pa, nvp);
	const int nb = countPolyVerts(pb, nvp);
	// If the merged polygon would be too big, do not merge.
	if (na+nb-2 > nvp)
		return -1;
	// Check if the polygons share an edge.
	// 首先检查有没有共同的边
	ea = -1;
	eb = -1;
	for (int i = 0; i < na; ++i) {
		unsigned short va0 = pa[i];
		unsigned short va1 = pa[(i+1) % na];
		if (va0 > va1)
			rcSwap(va0, va1);
		for (int j = 0; j < nb; ++j) {
			unsigned short vb0 = pb[j];
			unsigned short vb1 = pb[(j+1) % nb];
			if (vb0 > vb1)
				rcSwap(vb0, vb1);
			if (va0 == vb0 && va1 == vb1) {
				ea = i;
				eb = j;
				break;
			}
		}
	}
	// No common edge, cannot merge.
	if (ea == -1 || eb == -1)
		return -1;
	
	// Check to see if the merged polygon would be convex.
	// 检查合并后的多边形是不是凸多边形，通过检查合并边的两个点是不是在对角线的两边即可，这里通过改变对角线方向，转化为判断是不是都在左边
	unsigned short va, vb, vc;
	
	va = pa[(ea+na-1) % na];
	vb = pa[ea];
	vc = pb[(eb+2) % nb];
	if (!uleft(&verts[va*3], &verts[vb*3], &verts[vc*3]))
		return -1;
	
	va = pb[(eb+nb-1) % nb];
	vb = pb[eb];
	vc = pa[(ea+2) % na];
	if (!uleft(&verts[va*3], &verts[vb*3], &verts[vc*3]))
		return -1;
	
	va = pa[ea];
	vb = pa[(ea+1)%na];
	
	int dx = (int)verts[va*3+0] - (int)verts[vb*3+0];
	int dy = (int)verts[va*3+2] - (int)verts[vb*3+2];
	
	return dx*dx + dy*dy;
}
```

### 其它
- 到这里循环结束，还有一些工作需要完成。每个轮廓都划分为多边形，不过存在很多重复的顶点，需要清理。需要计算多边形的邻接情况。

```c++
	// Remove edge vertices.
	for (int i = 0; i < mesh.nverts; ++i) {
		if (vflags[i]) {
			if (!canRemoveVertex(ctx, mesh, (unsigned short)i))
				continue;
			removeVertex(ctx, mesh, (unsigned short)i, maxTris);
			// Remove vertex
			// Note: mesh.nverts is already decremented inside removeVertex()!
			// Fixup vertex flags
			for (int j = i; j < mesh.nverts; ++j)
				vflags[j] = vflags[j+1];
			--i;
		}
	}
	
	// Calculate adjacency.
	buildMeshAdjacency(mesh.polys, mesh.npolys, mesh.nverts, nvp);

	// Find portal edges
	if (mesh.borderSize > 0) {
		const int w = cset.width;
		const int h = cset.height;
		for (int i = 0; i < mesh.npolys; ++i) {
			unsigned short* p = &mesh.polys[i*2*nvp];
			for (int j = 0; j < nvp; ++j) {
				if (p[j] == RC_MESH_NULL_IDX) break;
				// Skip connected edges.
				if (p[nvp+j] != RC_MESH_NULL_IDX)
					continue;
				int nj = j+1;
				if (nj >= nvp || p[nj] == RC_MESH_NULL_IDX) nj = 0;
				const unsigned short* va = &mesh.verts[p[j]*3];
				const unsigned short* vb = &mesh.verts[p[nj]*3];

				if ((int)va[0] == 0 && (int)vb[0] == 0)
					p[nvp+j] = 0x8000 | 0;
				else if ((int)va[2] == h && (int)vb[2] == h)
					p[nvp+j] = 0x8000 | 1;
				else if ((int)va[0] == w && (int)vb[0] == w)
					p[nvp+j] = 0x8000 | 2;
				else if ((int)va[2] == 0 && (int)vb[2] == 0)
					p[nvp+j] = 0x8000 | 3;
			}
		}
	}

	// Just allocate the mesh flags array. The user is resposible to fill it.
	mesh.flags = (unsigned short*)rcAlloc(sizeof(unsigned short)*mesh.npolys, RC_ALLOC_PERM);
	memset(mesh.flags, 0, sizeof(unsigned short) * mesh.npolys);
	return true;
}
```
