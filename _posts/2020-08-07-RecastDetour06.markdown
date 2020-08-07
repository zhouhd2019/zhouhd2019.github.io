---
layout:     post
title:      "Contour"
subtitle:   "Recast Detour 06"
date:       2020-08-07 12:04:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - PathFinding
---

本篇对之前得到的区域进行处理，得到区域的轮廓
```c++
m_cset = rcAllocContourSet();
rcBuildContours(m_ctx, *m_chf, m_cfg.maxSimplificationError, m_cfg.maxEdgeLen, *m_cset);
```
主要流程都在rcBuildContours里面
### 标记边缘
- 第一部分是标记边缘，比较简单，遍历span，检查region标记，将连通性结果保存到函数变量flags
```c++
/// The raw contours will match the region outlines exactly. The @p maxError and @p maxEdgeLen
/// parameters control how closely the simplified contours will match the raw contours.
///
/// Simplified contours are generated such that the vertices for portals between areas match up.
/// (They are considered mandatory vertices.)
///
/// Setting @p maxEdgeLength to zero will disabled the edge length feature.
///
bool rcBuildContours(rcContext* ctx, rcCompactHeightfield& chf,
					 const float maxError, const int maxEdgeLen,
					 rcContourSet& cset, const int buildFlags)
{
	//......主要是初始化cset/flags，省略
	
	// Mark boundaries.
	for (int y = 0; y < h; ++y) {
		for (int x = 0; x < w; ++x) {
			const rcCompactCell& c = chf.cells[x+y*w];
			for (int i = (int)c.index, ni = (int)(c.index+c.count); i < ni; ++i) {
				unsigned char res = 0;
				const rcCompactSpan& s = chf.spans[i];
				if (!chf.spans[i].reg || (chf.spans[i].reg & RC_BORDER_REG)) {
					flags[i] = 0;
					continue;
				}
				for (int dir = 0; dir < 4; ++dir) {
					unsigned short r = 0;
					if (rcGetCon(s, dir) != RC_NOT_CONNECTED) {
						const int ax = x + rcGetDirOffsetX(dir);
						const int ay = y + rcGetDirOffsetY(dir);
						const int ai = (int)chf.cells[ax+ay*w].index + rcGetCon(s, dir);
						r = chf.spans[ai].reg;
					}
					if (r == chf.spans[i].reg)
						res |= (1 << dir);
				}
				flags[i] = res ^ 0xf; // Inverse, mark non connected edges.
			}
		}
	}
	// flags记录了span的连通性，0表示不可行走span或者区域内部，其它有1的标记位表示该方向不可行
```

### 循环处理
- 第二步是最重要的找出边缘，并进行简化，主要函数是walkContour和simplifyContour，在一个循环内对span进行处理
```c++
	rcIntArray verts(256);
	rcIntArray simplified(64);
	for (int y = 0; y < h; ++y) {
		for (int x = 0; x < w; ++x) {
			const rcCompactCell& c = chf.cells[x+y*w];
			for (int i = (int)c.index, ni = (int)(c.index+c.count); i < ni; ++i) {
				if (flags[i] == 0 || flags[i] == 0xf) {  // 不可行走，或者是区域内部，或者已经被遍历过了(walkContour会清空)
					flags[i] = 0;
					continue;
				}
				const unsigned short reg = chf.spans[i].reg;
				if (!reg || (reg & RC_BORDER_REG))
					continue;
				const unsigned char area = chf.areas[i];
				verts.resize(0);
				simplified.resize(0);
				
				walkContour(x, y, i, chf, flags, verts);
				simplifyContour(verts, simplified, maxError, maxEdgeLen, buildFlags);
				removeDegenerateSegments(simplified);
```
- 先看一下walkContour。获得轮廓的过程大概是从轮廓某个点出发，如果遇到边缘就记录，并顺时针旋转90度，继续往前；如果没有遇到边缘，就逆时针旋转90度；回到起点就结束，保存下来的点都是轮廓点。
```c++
static void walkContour(int x, int y, int i,
						rcCompactHeightfield& chf,
						unsigned char* flags, rcIntArray& points)
{
	// Choose the first non-connected edge
	unsigned char dir = 0;
	while ((flags[i] & (1 << dir)) == 0)
		dir++;
	unsigned char startDir = dir;
	int starti = i;
	const unsigned char area = chf.areas[i];
	
	int iter = 0;
	while (++iter < 40000) {
		if (flags[i] & (1 << dir)) {  // 在该方向不可行走，是一个轮廓点
			// Choose the edge corner
			bool isBorderVertex = false;
			bool isAreaBorder = false;
			int px = x;
			int py = getCornerHeight(x, y, i, dir, chf, isBorderVertex);
			int pz = y;
			switch(dir) {
				case 0: pz++; break;
				case 1: px++; pz++; break;
				case 2: px++; break;
			}
			int r = 0;
			const rcCompactSpan& s = chf.spans[i];
			if (rcGetCon(s, dir) != RC_NOT_CONNECTED) {
				const int ax = x + rcGetDirOffsetX(dir);
				const int ay = y + rcGetDirOffsetY(dir);
				const int ai = (int)chf.cells[ax+ay*chf.width].index + rcGetCon(s, dir);
				r = (int)chf.spans[ai].reg;
				if (area != chf.areas[ai])
					isAreaBorder = true;
			}
			if (isBorderVertex)
				r |= RC_BORDER_VERTEX;
			if (isAreaBorder)
				r |= RC_AREA_BORDER;
			points.push(px);
			points.push(py);
			points.push(pz);
			points.push(r);
			
			flags[i] &= ~(1 << dir); // Remove visited edges
			dir = (dir+1) & 0x3;  // Rotate CW  // 顺时针旋转90度再继续
		}
		else
		{  // 没有遇到边缘
			int ni = -1;
			const int nx = x + rcGetDirOffsetX(dir);
			const int ny = y + rcGetDirOffsetY(dir);
			const rcCompactSpan& s = chf.spans[i];
			if (rcGetCon(s, dir) != RC_NOT_CONNECTED)
			{
				const rcCompactCell& nc = chf.cells[nx+ny*chf.width];
				ni = (int)nc.index + rcGetCon(s, dir);
			}
			x = nx;
			y = ny;
			i = ni;
			dir = (dir+3) & 0x3;	// Rotate CCW  // 逆时针旋转90度
		}
		
		if (starti == i && startDir == dir)  // 回到起点
			break;
	}
}

```

- 上面获得的轮廓非常粗糙，接下来需要通过simplifyContour简化。
```c++
static void simplifyContour(rcIntArray& points, rcIntArray& simplified,
							const float maxError, const int maxEdgeLen, const int buildFlags)
{
	// Add initial points.
	bool hasConnections = false;
	for (int i = 0; i < points.size(); i += 4) {
		if ((points[i+3] & RC_CONTOUR_REG_MASK) != 0) {  // 提取到的是region id
			hasConnections = true;
			break;
		}
	}
	
	if (hasConnections) {
		// The contour has some portals to other regions.
		// Add a new point to every location where the region changes.
		for (int i = 0, ni = points.size()/4; i < ni; ++i) {  // 遍历所有轮廓点，ni是点的数量
			int ii = (i+1) % ni;
			const bool differentRegs = (points[i*4+3] & RC_CONTOUR_REG_MASK) != (points[ii*4+3] & RC_CONTOUR_REG_MASK);  // 点i和下一个点是不是同一个region
			const bool areaBorders = (points[i*4+3] & RC_AREA_BORDER) != (points[ii*4+3] & RC_AREA_BORDER);  // 点i和下一个点是不是处于area边缘
			if (differentRegs || areaBorders) {  // 这个点需要放到简化后的轮廓
				simplified.push(points[i*4+0]);
				simplified.push(points[i*4+1]);
				simplified.push(points[i*4+2]);
				simplified.push(i);
			}
		}
	}
	
	if (simplified.size() == 0) {
		// If there is no connections at all,
		// create some initial points for the simplification process.
		// Find lower-left and upper-right vertices of the contour.
		// 这一大片都是同一个region，没有找到合适的边界点，直接用左下和右上两个点作为起点。省略
	}
	
	// Add points until all raw points are within
	// error tolerance to the simplified shape.
	const int pn = points.size()/4;
	for (int i = 0; i < simplified.size()/4; ) {  // 遍历简化后的每条边
		int ii = (i+1) % (simplified.size()/4);
		int ax = simplified[i*4+0];
		int az = simplified[i*4+2];
		int ai = simplified[i*4+3];
		int bx = simplified[ii*4+0];
		int bz = simplified[ii*4+2];
		int bi = simplified[ii*4+3];

		// Find maximum deviation from the segment.

		// Traverse the segment in lexilogical order so that the
		// max deviation is calculated similarly when traversing
		// opposite segments.
		// 保证这条边两个点的相对大小，让每次遍历简单一点
		
		// Tessellate only outer edges or edges between areas.
		if ((points[ci*4+3] & RC_CONTOUR_REG_MASK) == 0 ||
			(points[ci*4+3] & RC_AREA_BORDER)) {
			while (ci != endi) {  // 遍历这条边中间所有点，找出离这条边最远的点
				float d = distancePtSeg(points[ci*4+0], points[ci*4+2], ax, az, bx, bz);
				if (d > maxd) {
					maxd = d;
					maxi = ci;
				}
				ci = (ci+cinc) % pn;
			}
		}
		
		// If the max deviation is larger than accepted error,
		// add new point, else continue to next segment.
		// 插入新的点到简化后的轮廓
		if (maxi != -1 && maxd > (maxError*maxError)) {
			// Add space for the new point.
			simplified.resize(simplified.size()+4);
			const int n = simplified.size()/4;
			for (int j = n-1; j > i; --j)
			{
				simplified[j*4+0] = simplified[(j-1)*4+0];
				simplified[j*4+1] = simplified[(j-1)*4+1];
				simplified[j*4+2] = simplified[(j-1)*4+2];
				simplified[j*4+3] = simplified[(j-1)*4+3];
			}
			// Add the point.
			simplified[(i+1)*4+0] = points[maxi*4+0];
			simplified[(i+1)*4+1] = points[maxi*4+1];
			simplified[(i+1)*4+2] = points[maxi*4+2];
			simplified[(i+1)*4+3] = maxi;
		}
		else {  // 这条边已经符合误差了，下一条边
			++i;
		}
	}
	
	// Split too long edges.省略
	
	for (int i = 0; i < simplified.size()/4; ++i)
	{
		// The edge vertex flag is take from the current raw point,
		// and the neighbour region is take from the next raw point.
		const int ai = (simplified[i*4+3]+1) % pn;
		const int bi = simplified[i*4+3];
		simplified[i*4+3] = (points[ai*4+3] & (RC_CONTOUR_REG_MASK|RC_AREA_BORDER)) | (points[bi*4+3] & RC_BORDER_VERTEX);
	}
	
}
```
- 上述步骤是循环内最重要的找出轮廓/简化轮廓，下面是简单的移除重复点
```c++
static void removeDegenerateSegments(rcIntArray& simplified) {
	// Remove adjacent vertices which are equal on xz-plane,
	// or else the triangulator will get confused.
	int npts = simplified.size()/4;
	for (int i = 0; i < npts; ++i) {
		int ni = next(i, npts);
		if (vequal(&simplified[i*4], &simplified[ni*4])) {
			// Degenerate segment, remove.
			for (int j = i; j < simplified.size()/4-1; ++j) {
				simplified[j*4+0] = simplified[(j+1)*4+0];
				simplified[j*4+1] = simplified[(j+1)*4+1];
				simplified[j*4+2] = simplified[(j+1)*4+2];
				simplified[j*4+3] = simplified[(j+1)*4+3];
			}
			simplified.resize(simplified.size()-4);
			npts--;
		}
	}
}
```

- 最后是将这次循环处理好的轮廓保存下来，主要是复制一下简化前后轮廓顶点
```c++
				// Store region->contour remap info.
				// Create contour.
				if (simplified.size()/4 >= 3) {
					// Allocate more contours.
					// This happens when a region has holes.
					// 轮廓数量超过了之前预计，重新分配一下指针数组。省略
					
					rcContour* cont = &cset.conts[cset.nconts++];
					cont->nverts = simplified.size()/4;
					cont->verts = (int*)rcAlloc(sizeof(int)*cont->nverts*4, RC_ALLOC_PERM);
					memcpy(cont->verts, &simplified[0], sizeof(int)*cont->nverts*4);
					// If the heightfield was build with bordersize, remove the offset.省略
					
					cont->nrverts = verts.size()/4;
					cont->rverts = (int*)rcAlloc(sizeof(int)*cont->nrverts*4, RC_ALLOC_PERM);
					memcpy(cont->rverts, &verts[0], sizeof(int)*cont->nrverts*4);
					// If the heightfield was build with bordersize, remove the offset.省略
					
					cont->reg = reg;
					cont->area = area;
				}
			}
		}
	}
```

### 收尾
-最后处理一下空洞。先通过面积计算判定轮廓有没有空洞，如果有就需要统计每个region的空洞，然后逐个进行合并。
```c++
	// Merge holes if needed.
	if (cset.nconts > 0) {
		// Calculate winding of all polygons.
		rcScopedDelete<char> winding((char*)rcAlloc(sizeof(char)*cset.nconts, RC_ALLOC_TEMP));
		int nholes = 0;
		for (int i = 0; i < cset.nconts; ++i) {
			rcContour& cont = cset.conts[i];
			// If the contour is wound backwards, it is a hole.
			winding[i] = calcAreaOfPolygon2D(cont.verts, cont.nverts) < 0 ? -1 : 1;
			if (winding[i] < 0)
				nholes++;
		}
		if (nholes > 0) {
			// Collect outline contour and holes contours per region.
			// We assume that there is one outline and multiple holes.
			// 统计每个region的空洞
			
			// Finally merge each regions holes into the outline.
			for (int i = 0; i < nregions; i++) {
				rcContourRegion& reg = regions[i];
				if (!reg.nholes) continue;
				if (reg.outline)
					mergeRegionHoles(ctx, reg);
			}
		}
	}
	return true;
}

```