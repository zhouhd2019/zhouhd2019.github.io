---
layout:     post
title:      "Smooth"
subtitle:   "Recast Detour 11"
date:       2020-08-13 10:50:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - PathFinding
---

上一回看了findPath，得到了一条由多个多边形组成的路径，接下来要根据这些多边形得到一条由多个线段组成的路径，即最后的结果。具体代码是NavMeshTesterTool::recalc中FOLLOW分支findPath接下来的部分
```c++
m_nsmoothPath = 0;
if (m_npolys)  // 之前的findPath有结果
{
	// Iterate over the path to find smooth path on the detail mesh surface.
	dtPolyRef polys[MAX_POLYS];
	memcpy(polys, m_polys, sizeof(dtPolyRef)*m_npolys); 
	int npolys = m_npolys;
	
	float iterPos[3], targetPos[3];
	m_navQuery->closestPointOnPoly(m_startRef, m_spos, iterPos, 0);
	m_navQuery->closestPointOnPoly(polys[npolys-1], m_epos, targetPos, 0);
	// 计算精确的起始和结束位置，主要是需要更新高度值(y)
	
	static const float STEP_SIZE = 0.5f;
	static const float SLOP = 0.01f;
	m_nsmoothPath = 0;
	dtVcopy(&m_smoothPath[m_nsmoothPath*3], iterPos);
	m_nsmoothPath++;
	
	// Move towards target a small advancement at a time until target reached or
	// when ran out of memory to store the path.
	// 多次循环，平滑路径
	while (npolys && m_nsmoothPath < MAX_SMOOTH)
	{
		// Find location to steer towards.
		float steerPos[3];
		unsigned char steerPosFlag;
		dtPolyRef steerPosRef;
		
		if (!getSteerTarget(m_navQuery, iterPos, targetPos, SLOP,
							polys, npolys, steerPos, steerPosFlag, steerPosRef))
			break;
```
首先第一步是getSteerTarget，不过getSteerTarget的第一步是findStraightPath，所以先看一下findStraightPath。从注释就可以知道这个函数通过string pulling来找出最短距离，可以说是本篇最核心的函数
```c++
/// This method peforms what is often called 'string pulling'.
dtStatus dtNavMeshQuery::findStraightPath(const float* startPos, const float* endPos,
										  const dtPolyRef* path, const int pathSize,
										  float* straightPath, unsigned char* straightPathFlags, dtPolyRef* straightPathRefs,
										  int* straightPathCount, const int maxStraightPath, const int options) const
{
	*straightPathCount = 0;
	dtStatus stat = 0;
	float closestStartPos[3];
	closestPointOnPolyBoundary(path[0], startPos, closestStartPos);  // 如果startPos在第一个多边形内，直接返回；否则返回最近边上面的点
	float closestEndPos[3];
	closestPointOnPolyBoundary(path[pathSize-1], endPos, closestEndPos);
	
	// 增加起点信息到结果集，会检查是否结束，相邻点是否重复等等
	stat = appendVertex(closestStartPos, DT_STRAIGHTPATH_START, path[0],
						straightPath, straightPathFlags, straightPathRefs,
						straightPathCount, maxStraightPath);
	if (stat != DT_IN_PROGRESS)
		return stat;
	
	if (pathSize > 1) {
		float portalApex[3], portalLeft[3], portalRight[3];
		dtVcopy(portalApex, closestStartPos);
		dtVcopy(portalLeft, portalApex);
		dtVcopy(portalRight, portalApex);
		int apexIndex = 0;
		int leftIndex = 0;
		int rightIndex = 0;
		unsigned char leftPolyType = 0;
		unsigned char rightPolyType = 0;
		dtPolyRef leftPolyRef = path[0];
		dtPolyRef rightPolyRef = path[0];
		
		for (int i = 0; i < pathSize; ++i) {  // 开始遍历路径的每个多边形
			float left[3], right[3];
			unsigned char fromType, toType;
			
			if (i+1 < pathSize) {
				// Next portal.
				// 注意这里获得了相邻多边形邻接边的左右端点
				if (dtStatusFailed(getPortalPoints(path[i], path[i+1], left, right, fromType, toType))) {
					// 特殊情况，path[i+1]是个无效的多边形，只能走到这里为止，返回部分结果
					return DT_SUCCESS | DT_PARTIAL_RESULT | ((*straightPathCount >= maxStraightPath) ? DT_BUFFER_TOO_SMALL : 0);
				}
				// If starting really close the portal, advance.
				if (i == 0) {
					float t;
					if (dtDistancePtSegSqr2D(portalApex, left, right, t) < dtSqr(0.001f))
						continue;
				}
			} else {  // 遍历完成
				// End of the path.
				dtVcopy(left, closestEndPos);
				dtVcopy(right, closestEndPos);
				fromType = toType = DT_POLYTYPE_GROUND;
			}

			// portalApex是当前寻路段的起点，遇到off-mesh或者大拐弯就会从拐点重新开始寻找路径，设定新portalApex
			// portalRight/portalLeft是当前寻路段portal的左右端点，注意不一定是多边形边界，如果寻路区域变宽就会是边界的一部分
			// right/left当前检查的portal多边形边界的左右端点

			// Right vertex.
			if (dtTriArea2D(portalApex, portalRight, right) <= 0.0f) {
				// 表明right在portalApex-portalRight直线左边或直线上
				// 即新的右点比原来的右点更偏向中心，会使得可行范围变窄
				if (dtVequal(portalApex, portalRight) || dtTriArea2D(portalApex, portalLeft, right) > 0.0f) {
					// portalApex和portalRight是同一个点，或者right在portalApex-portalLeft右边
					// 简单情况，新的右点没有越过左边
					dtVcopy(portalRight, right);
					rightPolyRef = (i+1 < pathSize) ? path[i+1] : 0;
					rightPolyType = toType;
					rightIndex = i;
				} else {
					// 新的右点越过了左边，需要保存当前拐点，再重新开始
					// Append portals along the current straight path segment.
					if (options & (DT_STRAIGHTPATH_AREA_CROSSINGS | DT_STRAIGHTPATH_ALL_CROSSINGS)) {
						stat = appendPortals(apexIndex, leftIndex, portalLeft, path,
											 straightPath, straightPathFlags, straightPathRefs,
											 straightPathCount, maxStraightPath, options);
						if (stat != DT_IN_PROGRESS)
							return stat;					
					}
				
					dtVcopy(portalApex, portalLeft);  // 以左边端点作为下次重新开始的起点
					apexIndex = leftIndex;
					
					unsigned char flags = 0;
					if (!leftPolyRef)
						flags = DT_STRAIGHTPATH_END;
					else if (leftPolyType == DT_POLYTYPE_OFFMESH_CONNECTION)
						flags = DT_STRAIGHTPATH_OFFMESH_CONNECTION;
					dtPolyRef ref = leftPolyRef;
					
					// Append or update vertex
					stat = appendVertex(portalApex, flags, ref,
										straightPath, straightPathFlags, straightPathRefs,
										straightPathCount, maxStraightPath);
					if (stat != DT_IN_PROGRESS)
						return stat;
					
					dtVcopy(portalLeft, portalApex);
					dtVcopy(portalRight, portalApex);
					leftIndex = apexIndex;
					rightIndex = apexIndex;
					// Restart
					i = apexIndex;  // 要从这个portal重新开始新的一段
					
					continue;
				}
			}
			
			// Left vertex.
			if (dtTriArea2D(portalApex, portalLeft, left) >= 0.0f) {
				// 左边和右边类似，省略......
			}
		}

		// 最后的portal可能是直路，要加一下
		// Append portals along the current straight path segment.
		if (options & (DT_STRAIGHTPATH_AREA_CROSSINGS | DT_STRAIGHTPATH_ALL_CROSSINGS)) {
			stat = appendPortals(apexIndex, pathSize-1, closestEndPos, path,
								 straightPath, straightPathFlags, straightPathRefs,
								 straightPathCount, maxStraightPath, options);
			if (stat != DT_IN_PROGRESS)
				return stat;
		}
	}
	// 尝试加终点
	stat = appendVertex(closestEndPos, DT_STRAIGHTPATH_END, 0,
						straightPath, straightPathFlags, straightPathRefs,
						straightPathCount, maxStraightPath);
	return DT_SUCCESS | ((*straightPathCount >= maxStraightPath) ? DT_BUFFER_TOO_SMALL : 0);
}
```

回到getSteerTarget，通过拉绳算法得到的路径保存在steerPath
```c++
static bool getSteerTarget(dtNavMeshQuery* navQuery, const float* startPos, const float* endPos,
						   const float minTargetDist,
						   const dtPolyRef* path, const int pathSize,
						   float* steerPos, unsigned char& steerPosFlag, dtPolyRef& steerPosRef,
						   float* outPoints = 0, int* outPointCount = 0)							 
{
	// Find steer target.
	static const int MAX_STEER_POINTS = 3;
	float steerPath[MAX_STEER_POINTS*3];
	unsigned char steerPathFlags[MAX_STEER_POINTS];
	dtPolyRef steerPathPolys[MAX_STEER_POINTS];
	int nsteerPath = 0;
	navQuery->findStraightPath(startPos, endPos, path, pathSize,
							   steerPath, steerPathFlags, steerPathPolys, &nsteerPath, MAX_STEER_POINTS);
	if (!nsteerPath)
		return false;
		
	// 保存中间结果，省略......
	// Find vertex far enough to steer to.
	// 找一个合适拐点
	int ns = 0;
	while (ns < nsteerPath) {
		// Stop at Off-Mesh link or when point is further than slop away.
		if ((steerPathFlags[ns] & DT_STRAIGHTPATH_OFFMESH_CONNECTION) || !inRange(&steerPath[ns*3], startPos, minTargetDist, 1000.0f))
			break;
		ns++;
	}
	// Failed to find good point to steer to.
	if (ns >= nsteerPath)
		return false;
	
	dtVcopy(steerPos, &steerPath[ns*3]);
	steerPos[1] = startPos[1];
	steerPosFlag = steerPathFlags[ns];
	steerPosRef = steerPathPolys[ns];
	return true;
}
```

再回到FOLLOW分支，这个时候已经找到一条由多个线段组成的路径，可以说关键部分已经完成，后面只是做一些整理。需要注意的是，每次只会走到拐点，通过这样的方法来平滑路径，所以这样的过程会进行多次
```c++
		bool endOfPath = (steerPosFlag & DT_STRAIGHTPATH_END) ? true : false;  // 拐点是不是终点
		bool offMeshConnection = (steerPosFlag & DT_STRAIGHTPATH_OFFMESH_CONNECTION) ? true : false;  // 拐点是不是offmesh点
		// Find movement delta.
		float delta[3], len;
		dtVsub(delta, steerPos, iterPos);  // iterPos当前起点
		len = dtMathSqrtf(dtVdot(delta, delta));
		// If the steer target is end of path or off-mesh link, do not move past the location.
		if ((endOfPath || offMeshConnection) && len < STEP_SIZE)
			len = 1;
		else
			len = STEP_SIZE / len;
		float moveTgt[3];
		dtVmad(moveTgt, iterPos, delta, len);  // 从iterPos往delta方向移动len
		
		// Move
		float result[3];
		dtPolyRef visited[16];
		int nvisited = 0;
		// 直线移动，和findPath进行类似的搜索，从iterPos走到moveTgt，记过放在result
		m_navQuery->moveAlongSurface(polys[0], iterPos, moveTgt, &m_filter,
										result, visited, &nvisited, 16);

		npolys = fixupCorridor(polys, npolys, MAX_POLYS, visited, nvisited);
		npolys = fixupShortcuts(polys, npolys, m_navQuery);
		// 上面是修正一些特殊情况，尽量剪除多余路径
		float h = 0;
		m_navQuery->getPolyHeight(polys[0], result, &h);
		result[1] = h;
		dtVcopy(iterPos, result);  // 这个中间点作为下次遍历的起点

		// Handle end of path and off-mesh links when close enough.
		if (endOfPath && inRange(iterPos, steerPos, SLOP, 1.0f)) {
			// Reached end of path.
			dtVcopy(iterPos, targetPos);
			if (m_nsmoothPath < MAX_SMOOTH) {
				dtVcopy(&m_smoothPath[m_nsmoothPath*3], iterPos);
				m_nsmoothPath++;
			}
			break;  // 到达终点，结束
		}
		else if (offMeshConnection && inRange(iterPos, steerPos, SLOP, 1.0f)) {
			// Reached off-mesh connection.
			float startPos[3], endPos[3];
			
			// Advance the path up to and over the off-mesh connection.
			dtPolyRef prevRef = 0, polyRef = polys[0];
			int npos = 0;
			while (npos < npolys && polyRef != steerPosRef) {
				prevRef = polyRef;
				polyRef = polys[npos];
				npos++;
			}
			for (int i = npos; i < npolys; ++i)
				polys[i-npos] = polys[i];
			npolys -= npos;
			
			// Handle the connection.
			dtStatus status = m_navMesh->getOffMeshConnectionPolyEndPoints(prevRef, polyRef, startPos, endPos);
			if (dtStatusSucceed(status)) {
				if (m_nsmoothPath < MAX_SMOOTH) {
					dtVcopy(&m_smoothPath[m_nsmoothPath*3], startPos);
					m_nsmoothPath++;
					// Hack to make the dotted path not visible during off-mesh connection.
					if (m_nsmoothPath & 1) {
						dtVcopy(&m_smoothPath[m_nsmoothPath*3], startPos);
						m_nsmoothPath++;
					}
				}
				// Move position at the other side of the off-mesh link.
				dtVcopy(iterPos, endPos);
				float eh = 0.0f;
				m_navQuery->getPolyHeight(polys[0], iterPos, &eh);
				iterPos[1] = eh;
			}
		}
		
		// Store results.
		if (m_nsmoothPath < MAX_SMOOTH) {
			dtVcopy(&m_smoothPath[m_nsmoothPath*3], iterPos);
			m_nsmoothPath++;
		}
	}
}
```
