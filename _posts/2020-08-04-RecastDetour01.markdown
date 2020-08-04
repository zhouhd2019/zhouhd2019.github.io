---
layout:     post
title:      "Rasterization"
subtitle:   "Recast Detour 01"
date:       2020-08-04 15:55:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - DataBase
---

以RecastDemo的Sample_TileMesh为例，简单看一下Recast Detour的部分源码。本节先看一下初始化和光栅化(体素化)过程。

### 初始化
- 选择Tile Mesh和模型，点击Build按钮，调用sample->handleBuild，这里的sample是Sample_TileMesh实例
```c++
bool Sample_TileMesh::handleBuild()
{
    //......
	dtFreeNavMesh(m_navMesh);
	m_navMesh = dtAllocNavMesh();
    //......释放老的NavMesh分配新的空间

	dtNavMeshParams params;
	//......设置dtNavMeshParams

	dtStatus status;
	status = m_navMesh->init(&params);

    status = m_navQuery->init(m_navMesh, 2048);

    if (m_buildAll)
		buildAllTiles();
	
	if (m_tool)
		m_tool->init(this);
	initToolStates(this);

	return true;
}
```
- dtNavMesh::init根据dtNavMeshParams进行初始化
```c++
dtStatus dtNavMesh::init(const dtNavMeshParams* params)
{
	//......
	m_tiles = (dtMeshTile*)dtAlloc(sizeof(dtMeshTile)*m_maxTiles, DT_ALLOC_PERM);
	m_posLookup = (dtMeshTile**)dtAlloc(sizeof(dtMeshTile*)*m_tileLutSize, DT_ALLOC_PERM);
    //......分配所有tile和查找表的空间，将tile组织成链表
    //......
	return DT_SUCCESS;
}
```
- dtNavMeshQuery::init，初始化寻路查询相关数据
```c++
dtStatus dtNavMeshQuery::init(const dtNavMesh* nav, const int maxNodes)
{
	//......
	m_nav = nav;
	if (!m_nodePool || m_nodePool->getMaxNodes() < maxNodes) {
		if (m_nodePool) {
			m_nodePool->~dtNodePool();
			dtFree(m_nodePool);
			m_nodePool = 0;
		}
		m_nodePool = new (dtAlloc(sizeof(dtNodePool), DT_ALLOC_PERM)) dtNodePool(maxNodes, dtNextPow2(maxNodes/4));
		//......placement new
	}
	else {
		m_nodePool->clear();
	}
	
	//同理初始化m_tinyNodePool和m_openList
	return DT_SUCCESS;
}
```
- 接下来是buildAllTiles，如果在界面选中了"Build All Tiles"，就会一次性创建所有tile，这里不进行展开，后面会看动态增删tile的代码
- 最后初始化SampleTool，这里实际上是NavMeshTileTool，init只是赋值了m_sample，initToolStates逐个调用SampleToolState的init函数，不是主要的逻辑，略过


### 光栅化
- 点击响应在NavMeshTileTool::handleClick，调用Sample_TileMesh::buildTile增加tile，调用Sample_TileMesh::removeTile删除tile
- buildTile
```c++
void Sample_TileMesh::buildTile(const float* pos)
{
	const float* bmin = m_geom->getNavMeshBoundsMin();
	const float* bmax = m_geom->getNavMeshBoundsMax();
	// 模型包围盒
	
	const float ts = m_tileSize*m_cellSize;
	const int tx = (int)((pos[0] - bmin[0]) / ts);
	const int ty = (int)((pos[2] - bmin[2]) / ts);
	// tile坐标
	
	m_lastBuiltTileBmin[0] = bmin[0] + tx*ts;
	m_lastBuiltTileBmin[1] = bmin[1];
	m_lastBuiltTileBmin[2] = bmin[2] + ty*ts;
	
	m_lastBuiltTileBmax[0] = bmin[0] + (tx+1)*ts;
	m_lastBuiltTileBmax[1] = bmax[1];
	m_lastBuiltTileBmax[2] = bmin[2] + (ty+1)*ts;
	// 记录tile的逻辑坐标，注意y轴没有划分tile，而是用了整个轴范围

	int dataSize = 0;
	unsigned char* data = buildTileMesh(tx, ty, m_lastBuiltTileBmin, m_lastBuiltTileBmax, dataSize);

	// Remove any previous data (navmesh owns and deletes the data).
	m_navMesh->removeTile(m_navMesh->getTileRefAt(tx,ty,0),0,0);

	// Add tile, or leave the location empty.
	if (data) {
		// Let the navmesh own the data.
		dtStatus status = m_navMesh->addTile(data,dataSize,DT_TILE_FREE_DATA,0,0);
		if (dtStatusFailed(status))
			dtFree(data);
	}
}
```
- 重点在Sample_TileMesh::buildTileMesh
```c++
unsigned char* Sample_TileMesh::buildTileMesh(const int tx, const int ty, const float* bmin, const float* bmax, int& dataSize)
{
	//......清理和初始化

	rcVcopy(m_cfg.bmin, bmin);
	rcVcopy(m_cfg.bmax, bmax);
	m_cfg.bmin[0] -= m_cfg.borderSize*m_cfg.cs;
	m_cfg.bmin[2] -= m_cfg.borderSize*m_cfg.cs;
	m_cfg.bmax[0] += m_cfg.borderSize*m_cfg.cs;
	m_cfg.bmax[2] += m_cfg.borderSize*m_cfg.cs;
	// 范围要比tile大一些，保证新的tile和邻接tile能够正确连接，以及边界的障碍物能够正确起作用

	// Allocate voxel heightfield where we rasterize our input data to.
	m_solid = rcAllocHeightfield();
	rcCreateHeightfield(m_ctx, *m_solid, m_cfg.width, m_cfg.height, m_cfg.bmin, m_cfg.bmax, m_cfg.cs, m_cfg.ch);
	
	// Allocate array that can hold triangle flags.
	// If you have multiple meshes you need to process, allocate
	// and array which can hold the max number of triangles you need to process.
	m_triareas = new unsigned char[chunkyMesh->maxTrisPerChunk];
	
	int cid[512];// TODO: Make grow when returning too many items.
	const int ncid = rcGetChunksOverlappingRect(chunkyMesh, tbmin, tbmax, cid, 512);
	
	m_tileTriCount = 0;
	for (int i = 0; i < ncid; ++i)
	{
		const rcChunkyTriMeshNode& node = chunkyMesh->nodes[cid[i]];
		const int* ctris = &chunkyMesh->tris[node.i*3];
		const int nctris = node.n;
		
		m_tileTriCount += nctris;
		
		memset(m_triareas, 0, nctris*sizeof(unsigned char));
		rcMarkWalkableTriangles(m_ctx, m_cfg.walkableSlopeAngle,
								verts, nverts, ctris, nctris, m_triareas);
		// 计算node所有三角面片的倾斜角，标记能否行走
		
		if (!rcRasterizeTriangles(m_ctx, verts, nverts, ctris, m_triareas, nctris, *m_solid, m_cfg.walkableClimb))
			return 0;
	}
	//......
```
- rcRasterizeTriangles，光栅化三角面片。遍历每个三角形，调用rasterizeTri进行光栅化
```c++
static bool rasterizeTri(const float* v0, const float* v1, const float* v2,
						 const unsigned char area, rcHeightfield& hf,
						 const float* bmin, const float* bmax,
						 const float cs, const float ics, const float ich,
						 const int flagMergeThr)
{
	//......初始化计算
	
	// Calculate the footprint of the triangle on the grid's y-axis
	int y0 = (int)((tmin[2] - bmin[2])*ics);
	int y1 = (int)((tmax[2] - bmin[2])*ics);
	y0 = rcClamp(y0, 0, h-1);
	y1 = rcClamp(y1, 0, h-1);
	// 当前三角形在z轴上的cell坐标，注意是z轴，注释里写的是平面上的y轴
	
	// Clip the triangle into all grid cells it touches.
	float buf[7*3*4];
	float *in = buf, *inrow = buf+7*3, *p1 = inrow+7*3, *p2 = p1+7*3;

	rcVcopy(&in[0], v0);
	rcVcopy(&in[1*3], v1);
	rcVcopy(&in[2*3], v2);
	int nvrow, nvIn = 3;
	
	for (int y = y0; y <= y1; ++y)
	{
		// Clip polygon to row. Store the remaining polygon as well
		// 先从z轴分割
		const float cz = bmin[2] + y*cs;
		dividePoly(in, nvIn, inrow, &nvrow, p1, &nvIn, cz+cs, 2);
```
- 先看一下分割函数dividePoly
```c++
// divides a convex polygons into two convex polygons on both sides of a line
static void dividePoly(const float* in, int nin,
					  float* out1, int* nout1,
					  float* out2, int* nout2,
					  float x, int axis)
{
	float d[12];
	for (int i = 0; i < nin; ++i)
		d[i] = x - in[i*3+axis];
	// 计算各个点在分割轴上坐标和分割处的差值

	int m = 0, n = 0;
	// 遍历输入多边形(分割过的就不只是三角形了)的每条边，从最后一条边开始
	for (int i = 0, j = nin-1; i < nin; j=i, ++i)
	{
		bool ina = d[j] >= 0;
		bool inb = d[i] >= 0;
		// 大于0表示在分割处的正方向，小于0表示在负方向，等于0表示就在分割处
		if (ina != inb)
		{  // 两点在分割处的两边
			float s = d[j] / (d[j] - d[i]);
			out1[m*3+0] = in[j*3+0] + (in[i*3+0] - in[j*3+0])*s;
			out1[m*3+1] = in[j*3+1] + (in[i*3+1] - in[j*3+1])*s;
			out1[m*3+2] = in[j*3+2] + (in[i*3+2] - in[j*3+2])*s;
			rcVcopy(out2 + n*3, out1 + m*3);  // 将分割点放到左和右结果集
			m++;
			n++;
			// add the i'th point to the right polygon. Do NOT add points that are on the dividing line
			// since these were already added above
			if (d[i] > 0)
			{
				rcVcopy(out1 + m*3, in + i*3);
				m++;
			}
			else if (d[i] < 0)
			{
				rcVcopy(out2 + n*3, in + i*3);
				n++;
			}
			// 将点i加入到左边或者右边，j不用管，后面会遍历到
		}
		else // same side
		{  // 该边的两点在分割处的同一边，将点i加到对应集合
			// add the i'th point to the right polygon. Addition is done even for points on the dividing line
			if (d[i] >= 0)
			{
				rcVcopy(out1 + m*3, in + i*3);
				m++;
				if (d[i] != 0)
					continue;
			}
			rcVcopy(out2 + n*3, in + i*3);
			n++;
		}
	}

	*nout1 = m;
	*nout2 = n;
}
```
- 回到rasterizeTri，之前将三角形根据z轴分隔，接下来进入内循环，根据x轴
```c++
		rcSwap(in, p1);  // 从小到大分割，未分割的边始终放在in，便于下次循环继续
		if (nvrow < 3) continue;
		// nvrow存放的是放在左边(坐标更小)的点的数量，内循环继续分割
		
		// find the horizontal bounds in the row
		float minX = inrow[0], maxX = inrow[0];
		for (int i=1; i<nvrow; ++i)
		{
			if (minX > inrow[i*3])	minX = inrow[i*3];
			if (maxX < inrow[i*3])	maxX = inrow[i*3];
		}
		int x0 = (int)((minX - bmin[0])*ics);
		int x1 = (int)((maxX - bmin[0])*ics);
		x0 = rcClamp(x0, 0, w-1);
		x1 = rcClamp(x1, 0, w-1);
		// 和之前类似，找出x轴的范围

		int nv, nv2 = nvrow;

		for (int x = x0; x <= x1; ++x)
		{
			// Clip polygon to column. store the remaining polygon as well
			const float cx = bmin[0] + x*cs;
			dividePoly(inrow, nv2, p1, &nv, p2, &nv2, cx+cs, 0);
			// 分割x轴
			rcSwap(inrow, p2);
			if (nv < 3) continue;
			
			// Calculate min and max of the span.
			float smin = p1[1], smax = p1[1];
			for (int i = 1; i < nv; ++i)
			{
				smin = rcMin(smin, p1[i*3+1]);
				smax = rcMax(smax, p1[i*3+1]);
			}
			smin -= bmin[1];
			smax -= bmin[1];
			// y轴范围

			// Skip the span if it is outside the heightfield bbox
			if (smax < 0.0f) continue;
			if (smin > by) continue;
			// Clamp the span to the heightfield bbox.
			if (smin < 0.0f) smin = 0;
			if (smax > by) smax = by;
			
			// Snap the span to the heightfield height grid.
			unsigned short ismin = (unsigned short)rcClamp((int)floorf(smin * ich), 0, RC_SPAN_MAX_HEIGHT);
			unsigned short ismax = (unsigned short)rcClamp((int)ceilf(smax * ich), (int)ismin+1, RC_SPAN_MAX_HEIGHT);
			
			if (!addSpan(hf, x, y, ismin, ismax, area, flagMergeThr))
				return false;
		}
	}

	return true;
}
```
- 经过z轴和x轴的分割，现在需要对y轴进行处理，执行addSpan
```c++
static bool addSpan(rcHeightfield& hf, const int x, const int y,
					const unsigned short smin, const unsigned short smax,
					const unsigned char area, const int flagMergeThr)
{
	// 注意x和y是目前循环分割到的cell坐标，smin和smax是y轴范围，area是可行走标记，flagMergeThr是行走的最大倾斜角
	int idx = x + y*hf.width;
	// 同一xz位置可能有多个三角形，经过分割后都会放在对应的hf.spans[idx]，组成链表
	
	rcSpan* s = allocSpan(hf);
	if (!s)
		return false;
	s->smin = smin;
	s->smax = smax;
	s->area = area;
	s->next = 0;
	
	// Empty cell, add the first span.
	if (!hf.spans[idx])
	{
		hf.spans[idx] = s;
		return true;
	}
	rcSpan* prev = 0;
	rcSpan* cur = hf.spans[idx];
	
	// Insert and merge spans.
	while (cur)
	{
		if (cur->smin > s->smax)
		{
			// Current span is further than the new span, break.
			break;
		}
		else if (cur->smax < s->smin)
		{
			// Current span is before the new span advance.
			prev = cur;
			cur = cur->next;
		}
		else
		{
			// Merge spans.
			if (cur->smin < s->smin)
				s->smin = cur->smin;
			if (cur->smax > s->smax)
				s->smax = cur->smax;
			
			// Merge flags.
			if (rcAbs((int)s->smax - (int)cur->smax) <= flagMergeThr)
				s->area = rcMax(s->area, cur->area);
				// 两个span的顶部足够近，更新可行走标记
			
			// Remove current span.
			rcSpan* next = cur->next;
			freeSpan(hf, cur);
			if (prev)
				prev->next = next;
			else
				hf.spans[idx] = next;
			cur = next;
		}
	}
	
	// Insert new span.
	if (prev)
	{
		s->next = prev->next;
		prev->next = s;
	}
	else
	{
		s->next = hf.spans[idx];
		hf.spans[idx] = s;
	}
	// 从底到顶的链表

	return true;
}
```

### 总结
上述步骤完成了建立tile的第一步，rcRasterizeTriangles，遍历tile范围内的所有三角形，进行光栅化，得到的是rcHeightfield，将三角形在xz平面体素化，每个xz坐标对应一个span链表，每个span对应一个体素(准确来说是方形长条)