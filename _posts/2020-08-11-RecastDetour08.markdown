---
layout:     post
title:      "Detail Mesh"
subtitle:   "Recast Detour 08"
date:       2020-08-11 09:50:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - PathFinding
---

上一步将寻路区域分成了凸多边形，这样寻路网格的创建就基本完成了，后面是根据寻路需要进一步构建寻路网格。本节主要关注detail mesh的创建
```c++
// Build detail mesh.
rcAllocPolyMeshDetail();
rcBuildPolyMeshDetail(m_ctx, *m_pmesh, *m_chf, m_cfg.detailSampleDist, m_cfg.detailSampleMaxError, *m_dmesh);
```
先看一下detail mesh有什么内容。比较简单，只有子网格/顶点/三角形数据。
```c++
/// Contains triangle meshes that represent detailed height data associated 
/// with the polygons in its associated polygon mesh object.
/// @ingroup recast
struct rcPolyMeshDetail
{
	unsigned int* meshes;	///< The sub-mesh data. [Size: 4*#nmeshes] 
	float* verts;			///< The mesh vertices. [Size: 3*#nverts] 
	unsigned char* tris;	///< The mesh triangles. [Size: 4*#ntris] 
	int nmeshes;			///< The number of sub-meshes defined by #meshes.
	int nverts;				///< The number of vertices in #verts.
	int ntris;				///< The number of triangles in #tris.
};
```
接下来关注重点，rcBuildPolyMeshDetail
```c++
bool rcBuildPolyMeshDetail(rcContext* ctx, const rcPolyMesh& mesh, const rcCompactHeightfield& chf,
						   const float sampleDist, const float sampleMaxError,
						   rcPolyMeshDetail& dmesh)
{
	const int nvp = mesh.nvp;  // 单个多边形顶点上限
	const float cs = mesh.cs;
	const float ch = mesh.ch;
	const float* orig = mesh.bmin;
	const int borderSize = mesh.borderSize;
	const int heightSearchRadius = rcMax(1, (int)ceilf(mesh.maxEdgeError));
	
	rcIntArray edges(64);
	rcIntArray tris(512);
	rcIntArray arr(512);
	rcIntArray samples(512);
	float verts[256*3];
	rcHeightPatch hp;
	int nPolyVerts = 0;
	int maxhw = 0, maxhh = 0;
	
	rcScopedDelete<int> bounds((int*)rcAlloc(sizeof(int)*mesh.npolys*4, RC_ALLOC_TEMP));
	rcScopedDelete<float> poly((float*)rcAlloc(sizeof(float)*nvp*3, RC_ALLOC_TEMP));
	
	// Find max size for a polygon area.
	// 找出每个多边形的包围盒，同时需要得到最大包围盒的大小和寻路网格顶点数，省略......
	
	hp.data = (unsigned short*)rcAlloc(sizeof(unsigned short)*maxhw*maxhh, RC_ALLOC_TEMP);
	
	dmesh.nmeshes = mesh.npolys;  // submesh就是之前的多边形
	dmesh.meshes = (unsigned int*)rcAlloc(sizeof(unsigned int)*dmesh.nmeshes*4, RC_ALLOC_PERM);
	
	int vcap = nPolyVerts+nPolyVerts/2;
	int tcap = vcap*2;
	
	dmesh.nverts = 0;
	dmesh.verts = (float*)rcAlloc(sizeof(float)*vcap*3, RC_ALLOC_PERM);
	dmesh.ntris = 0;
	dmesh.tris = (unsigned char*)rcAlloc(sizeof(unsigned char)*tcap*4, RC_ALLOC_PERM);
	
	for (int i = 0; i < mesh.npolys; ++i) {
		const unsigned short* p = &mesh.polys[i*nvp*2];
		
		// Store polygon vertices for processing.
		int npoly = 0;
		for (int j = 0; j < nvp; ++j) {
			if(p[j] == RC_MESH_NULL_IDX) break;
			const unsigned short* v = &mesh.verts[p[j]*3];
			poly[j*3+0] = v[0]*cs;
			poly[j*3+1] = v[1]*ch;
			poly[j*3+2] = v[2]*cs;
			npoly++;
		}
		
		// Get the height data from the area of the polygon.
		hp.xmin = bounds[i*4+0];
		hp.ymin = bounds[i*4+2];
		hp.width = bounds[i*4+1]-bounds[i*4+0];
		hp.height = bounds[i*4+3]-bounds[i*4+2];
		getHeightData(ctx, chf, p, npoly, mesh.verts, borderSize, hp, arr, mesh.regs[i]);
		
		// Build detail mesh.
		int nverts = 0;
		if (!buildPolyDetail(ctx, poly, npoly,
							 sampleDist, sampleMaxError,
							 heightSearchRadius, chf, hp,
							 verts, nverts, tris,
							 edges, samples)) {
			return false;
		}
		
		// Move detail verts to world space.
		// Offset poly too, will be used to flag checking.
		// 转换顶点和多边形到世界坐标，省略......
		
		// Store detail submesh.
		const int ntris = tris.size()/4;
		
		dmesh.meshes[i*4+0] = (unsigned int)dmesh.nverts;  // 起始顶点索引
		dmesh.meshes[i*4+1] = (unsigned int)nverts;  // 顶点数量
		dmesh.meshes[i*4+2] = (unsigned int)dmesh.ntris;  // 起始三角形索引
		dmesh.meshes[i*4+3] = (unsigned int)ntris;  // 三角形数量
		
		// Store vertices, allocate more memory if necessary.
		// 空间不够，重新分配，省略......
		for (int j = 0; j < nverts; ++j) {
			dmesh.verts[dmesh.nverts*3+0] = verts[j*3+0];
			dmesh.verts[dmesh.nverts*3+1] = verts[j*3+1];
			dmesh.verts[dmesh.nverts*3+2] = verts[j*3+2];
			dmesh.nverts++;
		}
		
		// Store triangles, allocate more memory if necessary.
		// 空间不够，重新分配，省略......
		for (int j = 0; j < ntris; ++j) {
			const int* t = &tris[j*4];
			dmesh.tris[dmesh.ntris*4+0] = (unsigned char)t[0];
			dmesh.tris[dmesh.ntris*4+1] = (unsigned char)t[1];
			dmesh.tris[dmesh.ntris*4+2] = (unsigned char)t[2];
			dmesh.tris[dmesh.ntris*4+3] = getTriFlags(&verts[t[0]*3], &verts[t[1]*3], &verts[t[2]*3], poly, npoly);
			dmesh.ntris++;
		}
	}
	
	return true;
}
```
可见rcBuildPolyMeshDetail在一个大循环里面逐个处理寻路网格的每个多边形，主要处理包括getHeightData和buildPolyDetail

### getHeightData
- getHeightData简单来说就是先找到边界，从边界往其它地方扩散高度值
```c++
static void getHeightData(rcContext* ctx, const rcCompactHeightfield& chf,
						  const unsigned short* poly, const int npoly,
						  const unsigned short* verts, const int bs,
						  rcHeightPatch& hp, rcIntArray& queue,
						  int region)
{
	queue.resize(0);
	memset(hp.data, 0xff, sizeof(unsigned short)*hp.width*hp.height);
	bool empty = true;

	// We cannot sample from this poly if it was created from polys
	// of different regions. If it was then it could potentially be overlapping
	// with polys of that region and the heights sampled here could be wrong.
	if (region != RC_MULTIPLE_REGS) {
		// Copy the height from the same region, and mark region borders
		// as seed points to fill the rest.
		for (int hy = 0; hy < hp.height; hy++) {
			int y = hp.ymin + hy + bs;
			for (int hx = 0; hx < hp.width; hx++) {
				int x = hp.xmin + hx + bs;
				const rcCompactCell& c = chf.cells[x + y*chf.width];  // 还是常见的二维遍历cell和span
				for (int i = (int)c.index, ni = (int)(c.index + c.count); i < ni; ++i) {
					const rcCompactSpan& s = chf.spans[i];
					if (s.reg == region) {
						// Store height
						hp.data[hx + hy*hp.width] = s.y;  // 集中保存一下高度
						empty = false;

						// If any of the neighbours is not in same region,
						// add the current location as flood fill start
						bool border = false;
						for (int dir = 0; dir < 4; ++dir) {
							if (rcGetCon(s, dir) != RC_NOT_CONNECTED) {
								const int ax = x + rcGetDirOffsetX(dir);
								const int ay = y + rcGetDirOffsetY(dir);
								const int ai = (int)chf.cells[ax + ay*chf.width].index + rcGetCon(s, dir);
								const rcCompactSpan& as = chf.spans[ai];
								if (as.reg != region) {
									border = true;
									break;
								}
							}
						}
						if (border)  // 存储一下边界点，存储的是x/y坐标和span索引
							push3(queue, x, y, i);
						break;
					}
				}
			}
		}
	}
	
	// 特殊情况，省略......

	static const int RETRACT_SIZE = 256;
	int head = 0;
	
	// We assume the seed is centered in the polygon, so a BFS to collect
	// height data will ensure we do not move onto overlapping polygons and
	// sample wrong heights.
	while (head*3 < queue.size()) {
		int cx = queue[head*3+0];
		int cy = queue[head*3+1];
		int ci = queue[head*3+2];
		head++;
		if (head >= RETRACT_SIZE) {  // 避免占用空间过多
			head = 0;
			if (queue.size() > RETRACT_SIZE*3)
				memmove(&queue[0], &queue[RETRACT_SIZE*3], sizeof(int)*(queue.size()-RETRACT_SIZE*3));
			queue.resize(queue.size()-RETRACT_SIZE*3);
		}
		
		const rcCompactSpan& cs = chf.spans[ci];
		for (int dir = 0; dir < 4; ++dir) {
			if (rcGetCon(cs, dir) == RC_NOT_CONNECTED) continue;
			const int ax = cx + rcGetDirOffsetX(dir);
			const int ay = cy + rcGetDirOffsetY(dir);
			const int hx = ax - hp.xmin - bs;
			const int hy = ay - hp.ymin - bs;
			if ((unsigned int)hx >= (unsigned int)hp.width || (unsigned int)hy >= (unsigned int)hp.height)
				continue;
			if (hp.data[hx + hy*hp.width] != RC_UNSET_HEIGHT)
				continue;
			const int ai = (int)chf.cells[ax + ay*chf.width].index + rcGetCon(cs, dir);
			const rcCompactSpan& as = chf.spans[ai];
			
			hp.data[hx + hy*hp.width] = as.y;
			push3(queue, ax, ay, ai);  // 继续往周围扩散高度
		}
	}
}
```

### buildPolyDetail
- 上一篇将寻路网格多边形化，但那是初步产生的，这一步的detail mesh，是将之前的网格进一步细化，直到在指定的误差内。主要分为两步，第一步是在多边形边上新增顶点，第二步时在多边形内新增顶点。将顶点集转为网格用的是triangulateHull和delaunayHull，这里不作展开
```c++
static bool buildPolyDetail(rcContext* ctx, const float* in, const int nin,
							const float sampleDist, const float sampleMaxError,
							const int heightSearchRadius, const rcCompactHeightfield& chf,
							const rcHeightPatch& hp, float* verts, int& nverts,
							rcIntArray& tris, rcIntArray& edges, rcIntArray& samples)
{
	static const int MAX_VERTS = 127;
	static const int MAX_TRIS = 255;	// Max tris for delaunay is 2n-2-k (n=num verts, k=num hull verts).
	static const int MAX_VERTS_PER_EDGE = 32;
	float edge[(MAX_VERTS_PER_EDGE+1)*3];
	int hull[MAX_VERTS];
	int nhull = 0;
	nverts = 0;
	
	for (int i = 0; i < nin; ++i)
		rcVcopy(&verts[i*3], &in[i*3]);
	nverts = nin;
	
	edges.resize(0);
	tris.resize(0);
	const float cs = chf.cs;
	const float ics = 1.0f/cs;
	// Calculate minimum extents of the polygon based on input data.
	float minExtent = polyMinExtent(verts, nverts);
	// Tessellate outlines.
	// This is done in separate pass in order to ensure
	// seamless height values across the ply boundaries.
	if (sampleDist > 0) {
		for (int i = 0, j = nin-1; i < nin; j=i++) {  // 常见的遍历所有边
			const float* vj = &in[j*3];
			const float* vi = &in[i*3];
			bool swapped = false;
			// Make sure the segments are always handled in same order
			// using lexological sort or else there will be seams.
			if (fabsf(vj[0]-vi[0]) < 1e-6f) {
				if (vj[2] > vi[2]) {
					rcSwap(vj,vi);
					swapped = true;
				}
			}
			else {
				if (vj[0] > vi[0]) {
					rcSwap(vj,vi);
					swapped = true;
				}
			}
			// Create samples along the edge.
			float dx = vi[0] - vj[0];
			float dy = vi[1] - vj[1];
			float dz = vi[2] - vj[2];
			float d = sqrtf(dx*dx + dz*dz);
			int nn = 1 + (int)floorf(d/sampleDist);
			if (nn >= MAX_VERTS_PER_EDGE) nn = MAX_VERTS_PER_EDGE-1;
			if (nverts+nn >= MAX_VERTS)
				nn = MAX_VERTS-1-nverts;
			for (int k = 0; k <= nn; ++k) {
				float u = (float)k/(float)nn;
				float* pos = &edge[k*3];
				pos[0] = vj[0] + dx*u;
				pos[1] = vj[1] + dy*u;
				pos[2] = vj[2] + dz*u;
				pos[1] = getHeight(pos[0],pos[1],pos[2], cs, ics, chf.ch, heightSearchRadius, hp)*chf.ch;  // 插值获得高度
			}
			// Simplify samples.
			int idx[MAX_VERTS_PER_EDGE] = {0,nn};
			int nidx = 2;
			// 尝试增加足够多的顶点，使得距离相关值在误差范围内
			for (int k = 0; k < nidx-1; ) {
				const int a = idx[k];
				const int b = idx[k+1];
				const float* va = &edge[a*3];
				const float* vb = &edge[b*3];
				// Find maximum deviation along the segment.
				float maxd = 0;
				int maxi = -1;
				for (int m = a+1; m < b; ++m) {
					float dev = distancePtSeg(&edge[m*3],va,vb);
					if (dev > maxd) {
						maxd = dev;
						maxi = m;
					}
				}
				// If the max deviation is larger than accepted error,
				// add new point, else continue to next segment.
				if (maxi != -1 && maxd > rcSqr(sampleMaxError)) {
					for (int m = nidx; m > k; --m)
						idx[m] = idx[m-1];
					idx[k+1] = maxi;
					nidx++;
				}
				else
					++k;
			}
			
			hull[nhull++] = j;
			// Add new vertices.
			// 应用刚才的结果，真正执行插入新顶点
			if (swapped) {
				for (int k = nidx-2; k > 0; --k) {
					rcVcopy(&verts[nverts*3], &edge[idx[k]*3]);
					hull[nhull++] = nverts;
					nverts++;
				}
			}
			else {
				for (int k = 1; k < nidx-1; ++k) {
					rcVcopy(&verts[nverts*3], &edge[idx[k]*3]);
					hull[nhull++] = nverts;
					nverts++;
				}
			}
		}
	}
	
	// If the polygon minimum extent is small (sliver or small triangle), do not try to add internal points.
	if (minExtent < sampleDist*2) {  // 网格很小，直接细分就可以了，不需要增加内部新顶点
		triangulateHull(nverts, verts, nhull, hull, tris);
		return true;
	}
	
	// Tessellate the base mesh.
	// We're using the triangulateHull instead of delaunayHull as it tends to
	// create a bit better triangulation for long thing triangles when there
	// are no internal points.
	triangulateHull(nverts, verts, nhull, hull, tris);
	if (sampleDist > 0) {
		// Create sample locations in a grid.
		float bmin[3], bmax[3];
		rcVcopy(bmin, in);
		rcVcopy(bmax, in);
		for (int i = 1; i < nin; ++i) {
			rcVmin(bmin, &in[i*3]);
			rcVmax(bmax, &in[i*3]);
		}
		int x0 = (int)floorf(bmin[0]/sampleDist);
		int x1 = (int)ceilf(bmax[0]/sampleDist);
		int z0 = (int)floorf(bmin[2]/sampleDist);
		int z1 = (int)ceilf(bmax[2]/sampleDist);
		samples.resize(0);
		for (int z = z0; z < z1; ++z) {  // 构建候选顶点
			for (int x = x0; x < x1; ++x) {
				float pt[3];
				pt[0] = x*sampleDist;
				pt[1] = (bmax[1]+bmin[1])*0.5f;
				pt[2] = z*sampleDist;
				// Make sure the samples are not too close to the edges.
				if (distToPoly(nin,in,pt) > -sampleDist/2) continue;
				samples.push(x);
				samples.push(getHeight(pt[0], pt[1], pt[2], cs, ics, chf.ch, heightSearchRadius, hp));
				samples.push(z);
				samples.push(0); // Not added
			}
		}
		
		// Add the samples starting from the one that has the most
		// error. The procedure stops when all samples are added
		// or when the max error is within treshold.
		const int nsamples = samples.size()/4;
		for (int iter = 0; iter < nsamples; ++iter) {
			if (nverts >= MAX_VERTS)
				break;
			// Find sample with most error.
			float bestpt[3] = {0,0,0};
			float bestd = 0;
			int besti = -1;
			for (int i = 0; i < nsamples; ++i) {  // 再遍历一次，找到误差最大的
				const int* s = &samples[i*4];
				if (s[3]) continue; // skip added.
				float pt[3];
				// The sample location is jittered to get rid of some bad triangulations
				// which are cause by symmetrical data from the grid structure.
				pt[0] = s[0]*sampleDist + getJitterX(i)*cs*0.1f;
				pt[1] = s[1]*chf.ch;
				pt[2] = s[2]*sampleDist + getJitterY(i)*cs*0.1f;
				float d = distToTriMesh(pt, verts, nverts, &tris[0], tris.size()/4);
				if (d < 0) continue; // did not hit the mesh.
				if (d > bestd) {
					bestd = d;
					besti = i;
					rcVcopy(bestpt,pt);
				}
			}
			// If the max error is within accepted threshold, stop tesselating.
			if (bestd <= sampleMaxError || besti == -1)
				break;
			// Mark sample as added.
			samples[besti*4+3] = 1;
			// Add the new sample point.
			rcVcopy(&verts[nverts*3],bestpt);
			nverts++;
			// Create new triangulation.
			// TODO: Incremental add instead of full rebuild.
			edges.resize(0);
			tris.resize(0);
			delaunayHull(ctx, nverts, verts, nhull, hull, tris, edges);
		}
	}
	return true;
}
```
