---
layout:     post
title:      "Region"
subtitle:   "Recast Detour 05"
date:       2020-08-06 17:31:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - PathFinding
---

本篇主要进行区域的生成，简单来说就是根据某种算法，将类似的span划分成region，便于后续建立三角形寻路区域。
```c++
// Partition the heightfield so that we can use simple algorithm later to triangulate the walkable areas.
// There are 3 martitioning methods, each with some pros and cons:
// 1) Watershed partitioning
//   - the classic Recast partitioning
//   - creates the nicest tessellation
//   - usually slowest
//   - partitions the heightfield into nice regions without holes or overlaps
//   - the are some corner cases where this method creates produces holes and overlaps
//      - holes may appear when a small obstacles is close to large open area (triangulation can handle this)v
//      - overlaps may occur if you have narrow spiral corridors (i.e stairs), this make triangulation to fail
//   * generally the best choice if you precompute the navmesh, use this if you have large open areas
// 2) Monotone partioning
//   - fastest
//   - partitions the heightfield into regions without holes and overlaps (guaranteed)
//   - creates long thin polygons, which sometimes causes paths with detours
//   * use this if you want fast navmesh generation
// 3) Layer partitoining
//   - quite fast
//   - partitions the heighfield into non-overlapping regions
//   - relies on the triangulation code to cope with holes (thus slower than monotone partitioning)
//   - produces better triangles than monotone partitioning
//   - does not have the corner cases of watershed partitioning
//   - can be slow and create a bit ugly tessellation (still better than monotone)
//     if you have large open areas with small obstacles (not a problem if you use tiles)
//   * good choice to use for tiled navmesh with medium and small sized tiles
```
算法有三种，分别是Watershed/Monotone/Layer，具体优劣代码注释做了比较清晰的说明。实际代码如下。
```c++
if (m_partitionType == SAMPLE_PARTITION_WATERSHED)
{
	// Prepare for region partitioning, by calculating distance field along the walkable surface.
	rcBuildDistanceField(m_ctx, *m_chf);
	// Partition the walkable surface into simple regions without holes.
	rcBuildRegions(m_ctx, *m_chf, m_cfg.borderSize, m_cfg.minRegionArea, m_cfg.mergeRegionArea);
}
else if (m_partitionType == SAMPLE_PARTITION_MONOTONE)
{
	// Partition the walkable surface into simple regions without holes.
	// Monotone partitioning does not need distancefield.
	rcBuildRegionsMonotone(m_ctx, *m_chf, m_cfg.borderSize, m_cfg.minRegionArea, m_cfg.mergeRegionArea);
}
else // SAMPLE_PARTITION_LAYERS
{
	// Partition the walkable surface into simple regions without holes.
	rcBuildLayerRegions(m_ctx, *m_chf, m_cfg.borderSize, m_cfg.minRegionArea);
}
```

### Watershed
- Watershed首先要执行rcBuildDistanceFiels。具体代码比较简单，调用calculateDistanceField计算距离，进行模糊，最后存储结果
```c++
/// This is usually the second to the last step in creating a fully built
/// compact heightfield.  This step is required before regions are built
/// using #rcBuildRegions or #rcBuildRegionsMonotone.
/// 
/// After this step, the distance data is available via the rcCompactHeightfield::maxDistance
/// and rcCompactHeightfield::dist fields.
///
/// @see rcCompactHeightfield, rcBuildRegions, rcBuildRegionsMonotone
bool rcBuildDistanceField(rcContext* ctx, rcCompactHeightfield& chf) {
	if (chf.dist) {
		rcFree(chf.dist);
		chf.dist = 0;
	}
	unsigned short* src = (unsigned short*)rcAlloc(sizeof(unsigned short)*chf.spanCount, RC_ALLOC_TEMP);
	unsigned short* dst = (unsigned short*)rcAlloc(sizeof(unsigned short)*chf.spanCount, RC_ALLOC_TEMP);
	
	unsigned short maxDist = 0;
	calculateDistanceField(chf, src, maxDist);
	chf.maxDistance = maxDist;
	if (boxBlur(chf, 1, src, dst) != src)
		rcSwap(src, dst);
	chf.dist = src;

	rcFree(dst);
	return true;
}
```
- calculateDistanceField和之前的erode过程基本一致，也是先找出边缘的span，接着从左下到右上遍历span，再从右上到左下遍历，计算出每个span到边缘的距离，另外还需要计算span的最大距离
- boxBlur是常见的模糊，3X3的卷积
- 接下来就是最重要的rcBuildRegions
```c++
/// Non-null regions will consist of connected, non-overlapping walkable spans that form a single contour.
/// Contours will form simple polygons.
/// 
/// If multiple regions form an area that is smaller than @p minRegionArea, then all spans will be
/// re-assigned to the zero (null) region.
/// 
/// Watershed partitioning can result in smaller than necessary regions, especially in diagonal corridors. 
/// @p mergeRegionArea helps reduce unecessarily small regions.
/// 
/// See the #rcConfig documentation for more information on the configuration parameters.
/// 
/// The region data will be available via the rcCompactHeightfield::maxRegions
/// and rcCompactSpan::reg fields.
/// 
/// @warning The distance field must be created using #rcBuildDistanceField before attempting to build regions.
/// 
/// @see rcCompactHeightfield, rcCompactSpan, rcBuildDistanceField, rcBuildRegionsMonotone, rcConfig
bool rcBuildRegions(rcContext* ctx, rcCompactHeightfield& chf,
					const int borderSize, const int minRegionArea, const int mergeRegionArea)
{
	const int w = chf.width;
	const int h = chf.height;
	rcScopedDelete<unsigned short> buf((unsigned short*)rcAlloc(sizeof(unsigned short)*chf.spanCount*4, RC_ALLOC_TEMP));

	const int LOG_NB_STACKS = 3;
	const int NB_STACKS = 1 << LOG_NB_STACKS;
	rcIntArray lvlStacks[NB_STACKS];
	for (int i=0; i<NB_STACKS; ++i)
		lvlStacks[i].resize(1024);

	rcIntArray stack(1024);
	rcIntArray visited(1024);
	
	unsigned short* srcReg = buf;
	unsigned short* srcDist = buf+chf.spanCount;
	unsigned short* dstReg = buf+chf.spanCount*2;
	unsigned short* dstDist = buf+chf.spanCount*3;
	memset(srcReg, 0, sizeof(unsigned short)*chf.spanCount);
	memset(srcDist, 0, sizeof(unsigned short)*chf.spanCount);
	
	unsigned short regionId = 1;
	unsigned short level = (chf.maxDistance+1) & ~1;  // 转为下一个偶数
	const int expandIters = 8;

	//......标记borderSize指定的四条边范围为边缘，略
	
	int sId = -1;
	while (level > 0) {
		level = level >= 2 ? level-2 : 0;
		sId = (sId+1) & (NB_STACKS-1);

		if (sId == 0)  // 刚开始，还没有排过序
			sortCellsByLevel(level, chf, srcReg, NB_STACKS, lvlStacks, 1);
		else   // 直接将之前没有处理的放到当前level的栈
			appendStacks(lvlStacks[sId-1], lvlStacks[sId], srcReg); // copy left overs from last level

		// Expand current regions until no empty connected cells found.
		if (expandRegions(expandIters, level, chf, srcReg, srcDist, dstReg, dstDist, lvlStacks[sId], false) != srcReg) {
			rcSwap(srcReg, dstReg);
			rcSwap(srcDist, dstDist);
		}

		// Mark new regions with IDs.
		for (int j = 0; j<lvlStacks[sId].size(); j += 3) {
			int x = lvlStacks[sId][j];
			int y = lvlStacks[sId][j+1];
			int i = lvlStacks[sId][j+2];
			if (i >= 0 && srcReg[i] == 0) {
				if (floodRegion(x, y, i, level, regionId, chf, srcReg, srcDist, stack)) {
					regionId++;
				}
			}
		}
	}
	// ......略
```

##### watershed循环
- rcBuildRegions第一步是通过循环对span进行watershed。首先要对span根据dist(除以2得到当前level，再求与起始level之差)，进行桶排序。如果已经排序过了，调用appendStacks即可
```c++
static void sortCellsByLevel(unsigned short startLevel,
							  rcCompactHeightfield& chf,
							  unsigned short* srcReg,
							  unsigned int nbStacks, rcIntArray* stacks,
							  unsigned short loglevelsPerStack) // the levels per stack (2 in our case) as a bit shift
{

	startLevel = startLevel >> loglevelsPerStack;
	// put all cells in the level range into the appropriate stacks
	for (int y = 0; y < h; ++y) {
		for (int x = 0; x < w; ++x) {
			const rcCompactCell& c = chf.cells[x+y*w];
			for (int i = (int)c.index, ni = (int)(c.index+c.count); i < ni; ++i) {
				if (chf.areas[i] == RC_NULL_AREA || srcReg[i] != 0)
					continue;
				int level = chf.dist[i] >> loglevelsPerStack;
				int sId = startLevel - level;  // dist越大，sId越小
				if (sId >= (int)nbStacks)
					continue;
				if (sId < 0)
					sId = 0;
				stacks[sId].push(x);
				stacks[sId].push(y);
				stacks[sId].push(i);
			}
		}
	}
}
```
- 得到当前level的排序结果后，就可以调用expandRegions
```c++
static unsigned short* expandRegions(int maxIter, unsigned short level,
									 rcCompactHeightfield& chf,
									 unsigned short* srcReg, unsigned short* srcDist,
									 unsigned short* dstReg, unsigned short* dstDist, 
									 rcIntArray& stack,
									 bool fillStack)
{
	const int w = chf.width;
	const int h = chf.height;

	if (fillStack) {
		//......走的是另一条分支
	}
	else // use cells in the input stack {
		// mark all cells which already have a region
		for (int j=0; j<stack.size(); j+=3) {
			int i = stack[j+2];
			if (srcReg[i] != 0)
				stack[j+2] = -1;
		}
	}

	int iter = 0;
	while (stack.size() > 0) {
		int failed = 0;
		memcpy(dstReg, srcReg, sizeof(unsigned short)*chf.spanCount);
		memcpy(dstDist, srcDist, sizeof(unsigned short)*chf.spanCount);
		
		for (int j = 0; j < stack.size(); j += 3) {
			int x = stack[j+0];
			int y = stack[j+1];
			int i = stack[j+2];
			if (i < 0) {  // 已经处理过了，不用再处理
				failed++;
				continue;
			}
			
			unsigned short r = srcReg[i];
			unsigned short d2 = 0xffff;
			const unsigned char area = chf.areas[i];
			const rcCompactSpan& s = chf.spans[i];
			for (int dir = 0; dir < 4; ++dir) {
				if (rcGetCon(s, dir) == RC_NOT_CONNECTED) continue;
				const int ax = x + rcGetDirOffsetX(dir);
				const int ay = y + rcGetDirOffsetY(dir);
				const int ai = (int)chf.cells[ax+ay*w].index + rcGetCon(s, dir);
				if (chf.areas[ai] != area) continue;
				if (srcReg[ai] > 0 && (srcReg[ai] & RC_BORDER_REG) == 0) {
					if ((int)srcDist[ai]+2 < (int)d2) {  // 遍历4方向，找到dist最小的span，记录它的region
						r = srcReg[ai];
						d2 = srcDist[ai]+2;
					}
				}
			}
			if (r) {
				stack[j+2] = -1; // mark as used
				dstReg[i] = r;
				dstDist[i] = d2;
			}
			else {
				failed++;
			}
		}
		
		// rcSwap source and dest.
		rcSwap(srcReg, dstReg);
		rcSwap(srcDist, dstDist);
		
		if (failed*3 == stack.size())  // 本轮中很多span都没有成功扩展，直接退出，丢给下一个level
			break;
		
		if (level > 0) {
			++iter;
			if (iter >= maxIter)  // 限制迭代次数，除了最后一轮需要尽量解决
				break;
		}
	}
	return srcReg;
}
```
- expandRegions以后，本level很多span的dist都变为一致，就差给它们标记regionId了。遍历本stack上的span，如果发现region还没有赋值，说明这是新增的region，就开始floodRegion
```c++
static bool floodRegion(int x, int y, int i,
						unsigned short level, unsigned short r,
						rcCompactHeightfield& chf,
						unsigned short* srcReg, unsigned short* srcDist,
						rcIntArray& stack)
{
	const int w = chf.width;
	const unsigned char area = chf.areas[i];
	
	// Flood fill mark region.
	stack.resize(0);
	stack.push((int)x);
	stack.push((int)y);
	stack.push((int)i);
	srcReg[i] = r;
	srcDist[i] = 0;
	
	unsigned short lev = level >= 2 ? level-2 : 0;
	int count = 0;
	
	while (stack.size() > 0) {
		int ci = stack.pop();
		int cy = stack.pop();
		int cx = stack.pop();
		const rcCompactSpan& cs = chf.spans[ci];
		
		// Check if any of the neighbours already have a valid region set.
		unsigned short ar = 0;
		for (int dir = 0; dir < 4; ++dir) {
			// 8 connected
			if (rcGetCon(cs, dir) != RC_NOT_CONNECTED) {
				const int ax = cx + rcGetDirOffsetX(dir);
				const int ay = cy + rcGetDirOffsetY(dir);
				const int ai = (int)chf.cells[ax+ay*w].index + rcGetCon(cs, dir);
				if (chf.areas[ai] != area)
					continue;
				unsigned short nr = srcReg[ai];
				if (nr & RC_BORDER_REG) // Do not take borders into account.
					continue;
				if (nr != 0 && nr != r) {
					ar = nr;
					break;
				}
				const rcCompactSpan& as = chf.spans[ai];
				const int dir2 = (dir+1) & 0x3;
				if (rcGetCon(as, dir2) != RC_NOT_CONNECTED) {
					const int ax2 = ax + rcGetDirOffsetX(dir2);
					const int ay2 = ay + rcGetDirOffsetY(dir2);
					const int ai2 = (int)chf.cells[ax2+ay2*w].index + rcGetCon(as, dir2);
					if (chf.areas[ai2] != area)
						continue;
					unsigned short nr2 = srcReg[ai2];
					if (nr2 != 0 && nr2 != r) {
						ar = nr2;
						break;
					}
				}				
			}
		}
		if (ar != 0) {  // 这个方向的扩展中止，相邻span有其它的regionId
			srcReg[ci] = 0;
			continue;
		}
		count++;
		
		// Expand neighbours.
		for (int dir = 0; dir < 4; ++dir) {  // 注意扩展检查时8方向，扩展时4方向，应该是保守起见
			if (rcGetCon(cs, dir) != RC_NOT_CONNECTED) {
				const int ax = cx + rcGetDirOffsetX(dir);
				const int ay = cy + rcGetDirOffsetY(dir);
				const int ai = (int)chf.cells[ax+ay*w].index + rcGetCon(cs, dir);
				if (chf.areas[ai] != area)
					continue;
				if (chf.dist[ai] >= lev && srcReg[ai] == 0) {
					srcReg[ai] = r;
					srcDist[ai] = 0;
					stack.push(ax);
					stack.push(ay);
					stack.push(ai);
				}
			}
		}
	}
	return count > 0;
}
```

##### watershed收尾
- 循环完成，每个level已经进行了watershed算法，大部分span都已经有了regionId，下面进行收尾，再进行一次expandRegions，并且对region进行筛选
```c++
	//......watershed循环
	// Expand current regions until no empty connected cells found.
	if (expandRegions(expandIters*8, 0, chf, srcReg, srcDist, dstReg, dstDist, stack, true) != srcReg)
	{
		rcSwap(srcReg, dstReg);
		rcSwap(srcDist, dstDist);
	}
	
	// Merge regions and filter out smalle regions.
	rcIntArray overlaps;
	chf.maxRegions = regionId;
	if (!mergeAndFilterRegions(ctx, minRegionArea, mergeRegionArea, chf.maxRegions, chf, srcReg, overlaps))
		return false;

	// If overlapping regions were found during merging, split those regions.
	if (overlaps.size() > 0)
	{
		ctx->log(RC_LOG_ERROR, "rcBuildRegions: %d overlapping regions.", overlaps.size());
	}

	// Write the result out.
	for (int i = 0; i < chf.spanCount; ++i)
		chf.spans[i].reg = srcReg[i];
	
	return true;
}
```
- mergeAndFilterRegions
```c++
static bool mergeAndFilterRegions(rcContext* ctx, int minRegionArea, int mergeRegionSize,
								  unsigned short& maxRegionId,
								  rcCompactHeightfield& chf,
								  unsigned short* srcReg, rcIntArray& overlaps)
{
	const int w = chf.width;
	const int h = chf.height;
	
	const int nreg = maxRegionId+1;
	rcRegion* regions = (rcRegion*)rcAlloc(sizeof(rcRegion)*nreg, RC_ALLOC_TEMP);
	// Construct regions
	for (int i = 0; i < nreg; ++i)
		new(&regions[i]) rcRegion((unsigned short)i);
	
	// Find edge of a region and find connections around the contour.
	// 找出region边缘
	for (int y = 0; y < h; ++y) {
		for (int x = 0; x < w; ++x) {
			const rcCompactCell& c = chf.cells[x+y*w];
			for (int i = (int)c.index, ni = (int)(c.index+c.count); i < ni; ++i) {
				unsigned short r = srcReg[i];
				if (r == 0 || r >= nreg)
					continue;
				rcRegion& reg = regions[r];
				reg.spanCount++;
				
				// Update floors.
				for (int j = (int)c.index; j < ni; ++j) {
					if (i == j) continue;
					unsigned short floorId = srcReg[j];
					if (floorId == 0 || floorId >= nreg)
						continue;
					if (floorId == r)
						reg.overlap = true;
					addUniqueFloorRegion(reg, floorId);
				}
				
				// Have found contour
				if (reg.connections.size() > 0)
					continue;
				reg.areaType = chf.areas[i];
				
				// Check if this cell is next to a border.
				int ndir = -1;
				for (int dir = 0; dir < 4; ++dir) {
					if (isSolidEdge(chf, srcReg, x, y, i, dir)) {
						ndir = dir;  // 在这个方向上span是边缘
						break;
					}
				}
				
				if (ndir != -1)
				{
					// The cell is at border.
					// Walk around the contour to find all the neighbours.
					// 找出边缘相邻的span，存放到reg.connections
					walkContour(x, y, i, ndir, chf, srcReg, reg.connections);
				}
			}
		}
	}

	// Remove too small regions.
	rcIntArray stack(32);
	rcIntArray trace(32);
	for (int i = 0; i < nreg; ++i) {
		rcRegion& reg = regions[i];
		if (reg.id == 0 || (reg.id & RC_BORDER_REG))
			continue;                       
		if (reg.spanCount == 0)
			continue;
		if (reg.visited)
			continue;
		
		// Count the total size of all the connected regions.
		// Also keep track of the regions connects to a tile border.
		bool connectsToBorder = false;
		int spanCount = 0;
		stack.resize(0);
		trace.resize(0);

		reg.visited = true;
		stack.push(i);
		
		while (stack.size()) {
			// Pop
			int ri = stack.pop();
			rcRegion& creg = regions[ri];
			spanCount += creg.spanCount;
			trace.push(ri);

			for (int j = 0; j < creg.connections.size(); ++j) {
				if (creg.connections[j] & RC_BORDER_REG) {
					connectsToBorder = true;
					continue;
				}
				rcRegion& neireg = regions[creg.connections[j]];  // 会将邻居region都一起考虑，如果一起考虑还不够大，后面就全部删除
				if (neireg.visited)
					continue;
				if (neireg.id == 0 || (neireg.id & RC_BORDER_REG))
					continue;
				// Visit
				stack.push(neireg.id);
				neireg.visited = true;
			}
		}
		
		// If the accumulated regions size is too small, remove it.
		// Do not remove areas which connect to tile borders
		// as their size cannot be estimated correctly and removing them
		// can potentially remove necessary areas.
		if (spanCount < minRegionArea && !connectsToBorder) {
			// Kill all visited regions.
			for (int j = 0; j < trace.size(); ++j) {
				regions[trace[j]].spanCount = 0;
				regions[trace[j]].id = 0;
			}
		}
	}
	
	// Merge too small regions to neighbour regions.
	int mergeCount = 0 ;
	do {
		mergeCount = 0;
		for (int i = 0; i < nreg; ++i) {
			rcRegion& reg = regions[i];
			if (reg.id == 0 || (reg.id & RC_BORDER_REG))  // 已被删除或者是边界
				continue;
			if (reg.overlap)  // 有重叠
				continue;
			if (reg.spanCount == 0)  // 已被删除
				continue;
			
			// Check to see if the region should be merged.
			if (reg.spanCount > mergeRegionSize && isRegionConnectedToBorder(reg))  // 足够大或者边界
				continue;
			
			// Small region with more than 1 connection.
			// Or region which is not connected to a border at all.
			// Find smallest neighbour region that connects to this one.
			int smallest = 0xfffffff;
			unsigned short mergeId = reg.id;
			// 找到面积最小的邻居，和它合并
			for (int j = 0; j < reg.connections.size(); ++j) {
				if (reg.connections[j] & RC_BORDER_REG) continue;
				rcRegion& mreg = regions[reg.connections[j]];
				if (mreg.id == 0 || (mreg.id & RC_BORDER_REG) || mreg.overlap) continue;
				if (mreg.spanCount < smallest &&
					canMergeWithRegion(reg, mreg) &&
					canMergeWithRegion(mreg, reg)) {
					smallest = mreg.spanCount;
					mergeId = mreg.id;
				}
			}
			// Found new id.
			// 找到了可以合并的邻居，合并
			if (mergeId != reg.id) {
				unsigned short oldId = reg.id;
				rcRegion& target = regions[mergeId];
				
				// Merge neighbours.
				if (mergeRegions(target, reg)) {
					// Fixup regions pointing to current region.
					for (int j = 0; j < nreg; ++j) {
						if (regions[j].id == 0 || (regions[j].id & RC_BORDER_REG)) continue;
						// If another region was already merged into current region
						// change the nid of the previous region too.
						if (regions[j].id == oldId)
							regions[j].id = mergeId;
						// Replace the current region with the new one if the
						// current regions is neighbour.
						replaceNeighbour(regions[j], oldId, mergeId);
					}
					mergeCount++;
				}
			}
		}
	}
	while (mergeCount > 0);
	
	// Compress region Ids.
	// Remap regions.
	// Return regions that we found to be overlapping.
	// ......省略

	return true;
}
```

### Monotone
- Monotone方式的实现函数是rcBuildRegionsMonotone，比较简单直观
```c++
bool rcBuildRegionsMonotone(rcContext* ctx, rcCompactHeightfield& chf,
							const int borderSize, const int minRegionArea, const int mergeRegionArea)
{
	// 初始化，省略
	// Mark border regions.省略
	rcIntArray prev(256);
	// Sweep one line at a time.
	for (int y = borderSize; y < h-borderSize; ++y) {
		// Collect spans from this row.
		prev.resize(id+1);
		memset(&prev[0],0,sizeof(int)*id);
		unsigned short rid = 1;
		
		for (int x = borderSize; x < w-borderSize; ++x) {
			const rcCompactCell& c = chf.cells[x+y*w];
			
			for (int i = (int)c.index, ni = (int)(c.index+c.count); i < ni; ++i) {
				const rcCompactSpan& s = chf.spans[i];
				//......遍历每个span，简单地在x/z两个方向扩展span，检查能不能扩展区域，省略
			}
		}
		
		// Create unique ID.
		// Remap IDs
		//......归纳扩展结果，省略
	}
	// Merge regions and filter out small regions.
	rcIntArray overlaps;
	chf.maxRegions = id;
	if (!mergeAndFilterRegions(ctx, minRegionArea, mergeRegionArea, chf.maxRegions, chf, srcReg, overlaps))
		return false;
	// Store the result out.
	for (int i = 0; i < chf.spanCount; ++i)
		chf.spans[i].reg = srcReg[i];
	return true;
}
```

### Layer
- 调用的是rcBuildLayerRegions，初步合并region的代码和Monotone的完全一致，不同的是最后归纳筛选region用的是mergeAndFilterLayerRegions，采用的是栈来逐渐扩张region。而mergeAndFilterRegions用的是描边法
```c++
static bool mergeAndFilterLayerRegions(rcContext* ctx, int minRegionArea,
									   unsigned short& maxRegionId,
									   rcCompactHeightfield& chf,
									   unsigned short* srcReg, rcIntArray& /*overlaps*/)
{
	const int w = chf.width;
	const int h = chf.height;
	const int nreg = maxRegionId+1;
	rcRegion* regions = (rcRegion*)rcAlloc(sizeof(rcRegion)*nreg, RC_ALLOC_TEMP);
	// Construct regions
	for (int i = 0; i < nreg; ++i)
		new(&regions[i]) rcRegion((unsigned short)i);
	
	// Find region neighbours and overlapping regions.
	rcIntArray lregs(32);
	for (int y = 0; y < h; ++y) {
		for (int x = 0; x < w; ++x) {
			const rcCompactCell& c = chf.cells[x+y*w];
			lregs.resize(0);
			for (int i = (int)c.index, ni = (int)(c.index+c.count); i < ni; ++i) {
				// Collect all region layers.
				// Update neighbours......				
			}
			// Update overlapping regions.......
		}
	}

	// Create 2D layers from regions.
	unsigned short layerId = 1;

	for (int i = 0; i < nreg; ++i)
		regions[i].id = 0;

	// Merge montone regions to create non-overlapping areas.
	rcIntArray stack(32);
	for (int i = 1; i < nreg; ++i) {
		rcRegion& root = regions[i];
		// Skip already visited.
		if (root.id != 0)
			continue;
		
		// Start search.
		root.id = layerId;

		stack.resize(0);
		stack.push(i);
		
		while (stack.size() > 0) {
			// Pop front
			rcRegion& reg = regions[stack[0]];
			for (int j = 0; j < stack.size()-1; ++j)
				stack[j] = stack[j+1];
			stack.resize(stack.size()-1);
			
			const int ncons = (int)reg.connections.size();
			for (int j = 0; j < ncons; ++j) {
				//......检查邻居，如果邻居不是重叠的，就将邻居加入region，并且加入栈，后面从这个邻居继续扩张
			}
		}
		
		layerId++;
	}
	
	// Remove small regions......省略
	// Compress region Ids.......省略
	// Remap regions.......省略
	
	return true;
}
```
