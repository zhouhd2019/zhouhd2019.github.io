---
layout:     post
title:      "Erode"
subtitle:   "Recast Detour 04"
date:       2020-08-05 14:57:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - PathFinding
---

第四部分是生成region前的准备，主要要根据寻路角色半径对span进行修改
### Erode
- 寻路需要考虑到agent的半径，在生成region之前，需要把不合法的cell去除。另外RecastDetour支持区域标记，就在erode后进行
```c++
// Erode the walkable area by agent radius.
if (!rcErodeWalkableArea(m_ctx, m_cfg.walkableRadius, *m_chf))
{
	m_ctx->log(RC_LOG_ERROR, "buildNavigation: Could not erode.");
	return 0;
}
// (Optional) Mark areas.
const ConvexVolume* vols = m_geom->getConvexVolumes();
for (int i  = 0; i < m_geom->getConvexVolumeCount(); ++i)
	rcMarkConvexPolyArea(m_ctx, vols[i].verts, vols[i].nverts, vols[i].hmin, vols[i].hmax, (unsigned char)vols[i].area, *m_chf);
```
- 首先看一下erode，比较简单，先找出边缘span，然后计算其它span到边缘的距离，最后根据所得距离，去除那些离边缘过近的span
```c++
/// Basically, any spans that are closer to a boundary or obstruction than the specified radius 
/// are marked as unwalkable.
///
/// This method is usually called immediately after the heightfield has been built.
///
/// @see rcCompactHeightfield, rcBuildCompactHeightfield, rcConfig::walkableRadius
bool rcErodeWalkableArea(rcContext* ctx, int radius, rcCompactHeightfield& chf)
{
	const int w = chf.width;
	const int h = chf.height;
		
	unsigned char* dist = (unsigned char*)rcAlloc(sizeof(unsigned char)*chf.spanCount, RC_ALLOC_TEMP);
	memset(dist, 0xff, sizeof(unsigned char)*chf.spanCount);
	
	// Mark boundary cells.
	for (int y = 0; y < h; ++y) {
		for (int x = 0; x < w; ++x) {
			const rcCompactCell& c = chf.cells[x+y*w];
			for (int i = (int)c.index, ni = (int)(c.index+c.count); i < ni; ++i) {
				if (chf.areas[i] == RC_NULL_AREA)  // span本身不可走
					dist[i] = 0;
				else {
					const rcCompactSpan& s = chf.spans[i];
					int nc = 0;
					for (int dir = 0; dir < 4; ++dir) {
						if (rcGetCon(s, dir) != RC_NOT_CONNECTED) {
							const int nx = x + rcGetDirOffsetX(dir);
							const int ny = y + rcGetDirOffsetY(dir);
							const int nidx = (int)chf.cells[nx+ny*w].index + rcGetCon(s, dir);
							if (chf.areas[nidx] != RC_NULL_AREA)
								nc++;
						}
					}
					// At least one missing neighbour.
					if (nc != 4) dist[i] = 0;
				}
			}
		}
	}
	// 这一步只完成了边缘span的标记，即span本身不可走，或者span有一个方向上没有邻居
	
	unsigned char nd;
	// Pass 1
	// 从左下到右上遍历
	for (int y = 0; y < h; ++y) {
		for (int x = 0; x < w; ++x) {
			const rcCompactCell& c = chf.cells[x+y*w];
			for (int i = (int)c.index, ni = (int)(c.index+c.count); i < ni; ++i) {
				const rcCompactSpan& s = chf.spans[i];
				
				if (rcGetCon(s, 0) != RC_NOT_CONNECTED) {
					// (-1,0)
					const int ax = x + rcGetDirOffsetX(0);
					const int ay = y + rcGetDirOffsetY(0);
					const int ai = (int)chf.cells[ax+ay*w].index + rcGetCon(s, 0);
					const rcCompactSpan& as = chf.spans[ai];
					nd = (unsigned char)rcMin((int)dist[ai]+2, 255);
					if (nd < dist[i]) dist[i] = nd;
					
					// (-1,-1)
					if (rcGetCon(as, 3) != RC_NOT_CONNECTED) {
						const int aax = ax + rcGetDirOffsetX(3);
						const int aay = ay + rcGetDirOffsetY(3);
						const int aai = (int)chf.cells[aax+aay*w].index + rcGetCon(as, 3);
						nd = (unsigned char)rcMin((int)dist[aai]+3, 255);
						if (nd < dist[i]) dist[i] = nd;
					}
				}
				if (rcGetCon(s, 3) != RC_NOT_CONNECTED) {
					// (0,-1)
					const int ax = x + rcGetDirOffsetX(3);
					const int ay = y + rcGetDirOffsetY(3);
					const int ai = (int)chf.cells[ax+ay*w].index + rcGetCon(s, 3);
					const rcCompactSpan& as = chf.spans[ai];
					nd = (unsigned char)rcMin((int)dist[ai]+2, 255);
					if (nd < dist[i]) dist[i] = nd;
					
					// (1,-1)
					if (rcGetCon(as, 2) != RC_NOT_CONNECTED) {
						const int aax = ax + rcGetDirOffsetX(2);
						const int aay = ay + rcGetDirOffsetY(2);
						const int aai = (int)chf.cells[aax+aay*w].index + rcGetCon(as, 2);
						nd = (unsigned char)rcMin((int)dist[aai]+3, 255);
						if (nd < dist[i]) dist[i] = nd;
					}
				}
			}
		}
	}
	
	// Pass 2
	// 从右上到左下遍历
	for (int y = h-1; y >= 0; --y) {
		for (int x = w-1; x >= 0; --x) {
			const rcCompactCell& c = chf.cells[x+y*w];
			for (int i = (int)c.index, ni = (int)(c.index+c.count); i < ni; ++i) {
				//......略
			}
		}
	}
	
	const unsigned char thr = (unsigned char)(radius*2);
	for (int i = 0; i < chf.spanCount; ++i)
		if (dist[i] < thr)
			chf.areas[i] = RC_NULL_AREA;  // 真正erode
	rcFree(dist);
	return true;
}
```

### Mark Convex Poly Area
- 用户可以手动画一篇区域，并指定区域的标记，在寻路的时候可以指定标记进行不同的请求，例如只走road标记的区域。rcMarkConvexPolyArea，注意不会在y轴有精确的计算（例如分割原有span）。
```c++
/// The value of spacial parameters are in world units.
/// 
/// The y-values of the polygon vertices are ignored. So the polygon is effectively 
/// projected onto the xz-plane at @p hmin, then extruded to @p hmax.
/// 
/// @see rcCompactHeightfield, rcMedianFilterWalkableArea
void rcMarkConvexPolyArea(rcContext* ctx, const float* verts, const int nverts,
						  const float hmin, const float hmax, unsigned char areaId,
						  rcCompactHeightfield& chf)
{
	//......计算得到自定义区域的包围盒minx/y/z和maxx/y/z，cell坐标

	for (int z = minz; z <= maxz; ++z) {
		for (int x = minx; x <= maxx; ++x) {
			const rcCompactCell& c = chf.cells[x+z*chf.width];
			for (int i = (int)c.index, ni = (int)(c.index+c.count); i < ni; ++i) {
				rcCompactSpan& s = chf.spans[i];
				if (chf.areas[i] == RC_NULL_AREA)
					continue;
				if ((int)s.y >= miny && (int)s.y <= maxy) {
					float p[3];
					p[0] = chf.bmin[0] + (x+0.5f)*chf.cs; 
					p[1] = 0;
					p[2] = chf.bmin[2] + (z+0.5f)*chf.cs; 
					if (pointInPoly(nverts, verts, p)) {  // span在自定义区域内
						chf.areas[i] = areaId;
					}
				}
			}
		}
	}
}
```
