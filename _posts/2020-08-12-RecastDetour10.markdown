---
layout:     post
title:      "findPath"
subtitle:   "Recast Detour 10"
date:       2020-08-12 16:40:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - PathFinding
---

之前看的主要是Recast，后面看一下detour。一次寻路的入口在sample->handleClick，这里会调用当前工具m_tool->handleClick，寻路工具的对应实现是NavMeshTesterTool::handleClick，实际上就是调用NavMeshTesterTool::recalc
```c++
if (m_sposSet)
	m_navQuery->findNearestPoly(m_spos, m_polyPickExt, &m_filter, &m_startRef, 0);
else
	m_startRef = 0;

if (m_eposSet)
	m_navQuery->findNearestPoly(m_epos, m_polyPickExt, &m_filter, &m_endRef, 0);
else
	m_endRef = 0;
```
函数一开始会检查寻路的开始和结束位置有没有设置，如果设置了会调用findNearestPoly来设置m_startRef或者m_endRef，这里省略过程，直接看寻路。NavMeshTesterTool支持很多模式，这里只看最常用的follow模式
```c++
m_navQuery->findPath(m_startRef, m_endRef, m_spos, m_epos, &m_filter, m_polys, &m_npolys, MAX_POLYS);
```
- 可以看到一上来就是findPath，先去看一下最重要的findPath，基本上就是A*，注意现在只是得到经过的多边形，没有形成多条线段组成的路径
```c++
/// If the end polygon cannot be reached through the navigation graph,
/// the last polygon in the path will be the nearest the end polygon.
///
/// If the path array is to small to hold the full result, it will be filled as 
/// far as possible from the start polygon toward the end polygon.
///
/// The start and end positions are used to calculate traversal costs. 
/// (The y-values impact the result.)
///
dtStatus dtNavMeshQuery::findPath(dtPolyRef startRef, dtPolyRef endRef,
								  const float* startPos, const float* endPos,
								  const dtQueryFilter* filter,
								  dtPolyRef* path, int* pathCount, const int maxPath) const
{
	// Validate input，省略......
	m_nodePool->clear();
	m_openList->clear();
	
	dtNode* startNode = m_nodePool->getNode(startRef);
	dtVcopy(startNode->pos, startPos);
	startNode->pidx = 0;
	startNode->cost = 0;
	startNode->total = dtVdist(startPos, endPos) * H_SCALE;
	startNode->id = startRef;
	startNode->flags = DT_NODE_OPEN;
	m_openList->push(startNode);  // 压入，经典的搜索准备
	
	dtNode* lastBestNode = startNode;
	float lastBestNodeCost = startNode->total;
	
	dtStatus status = DT_SUCCESS;
	while (!m_openList->empty()) {
		// Remove node from open list and put it in closed list.
		dtNode* bestNode = m_openList->pop();
		bestNode->flags &= ~DT_NODE_OPEN;
		bestNode->flags |= DT_NODE_CLOSED;
		
		// Reached the goal, stop searching.
		if (bestNode->id == endRef) {
			lastBestNode = bestNode;
			break;  // 不会找出最优路径
		}
		
		// Get current poly and tile.
		// The API input has been cheked already, skip checking internal data.
		const dtPolyRef bestRef = bestNode->id;
		const dtMeshTile* bestTile = 0;
		const dtPoly* bestPoly = 0;
		m_nav->getTileAndPolyByRefUnsafe(bestRef, &bestTile, &bestPoly);
		// Get parent poly and tile.
		dtPolyRef parentRef = 0;
		const dtMeshTile* parentTile = 0;
		const dtPoly* parentPoly = 0;
		if (bestNode->pidx)
			parentRef = m_nodePool->getNodeAtIdx(bestNode->pidx)->id;
		if (parentRef)
			m_nav->getTileAndPolyByRefUnsafe(parentRef, &parentTile, &parentPoly);

		// 开始遍历当前节点的相邻节点
		for (unsigned int i = bestPoly->firstLink; i != DT_NULL_LINK; i = bestTile->links[i].next) {
			dtPolyRef neighbourRef = bestTile->links[i].ref;
			// Skip invalid ids and do not expand back to where we came from.
			if (!neighbourRef || neighbourRef == parentRef)
				continue;
			
			// Get neighbour poly and tile.
			// The API input has been cheked already, skip checking internal data.
			const dtMeshTile* neighbourTile = 0;
			const dtPoly* neighbourPoly = 0;
			m_nav->getTileAndPolyByRefUnsafe(neighbourRef, &neighbourTile, &neighbourPoly);			
			
			if (!filter->passFilter(neighbourRef, neighbourTile, neighbourPoly))  // 这个是检查flag
				continue;

			// deal explicitly with crossing tile boundaries
			unsigned char crossSide = 0;
			if (bestTile->links[i].side != 0xff)
				crossSide = bestTile->links[i].side >> 1;

			// get the node
			dtNode* neighbourNode = m_nodePool->getNode(neighbourRef, crossSide);
			if (!neighbourNode) {
				status |= DT_OUT_OF_NODES;
				continue;
			}
			
			// If the node is visited the first time, calculate node position.
			// 计算走到邻接node经过的边，那条边的中点。后面都会用这个点来估算距离等等
			if (neighbourNode->flags == 0)
				getEdgeMidPoint(bestRef, bestPoly, bestTile, neighbourRef, neighbourPoly, neighbourTile, neighbourNode->pos);

			// Calculate cost and heuristic.
			float cost = 0;
			float heuristic = 0;
			
			// Special case for last node.
			if (neighbourRef == endRef) {
				const float curCost = filter->getCost(bestNode->pos, neighbourNode->pos,
													  parentRef, parentTile, parentPoly,
													  bestRef, bestTile, bestPoly,
													  neighbourRef, neighbourTile, neighbourPoly);
				const float endCost = filter->getCost(neighbourNode->pos, endPos,
													  bestRef, bestTile, bestPoly,
													  neighbourRef, neighbourTile, neighbourPoly,
													  0, 0, 0);
				cost = bestNode->cost + curCost + endCost;
				heuristic = 0;  // 找到一条可行路径，直接分成两部分来计算cost
			} else {  // 还没有走到终点，算bestNode到当前邻居节点的cost，加上起点到bestNode的cost，再加上邻居节点到终点的距离相关值
				const float curCost = filter->getCost(bestNode->pos, neighbourNode->pos,
													  parentRef, parentTile, parentPoly,
													  bestRef, bestTile, bestPoly,
													  neighbourRef, neighbourTile, neighbourPoly);
				cost = bestNode->cost + curCost;
				heuristic = dtVdist(neighbourNode->pos, endPos)*H_SCALE;
			}

			const float total = cost + heuristic;
			
			// The node is already in open list and the new result is worse, skip.
			if ((neighbourNode->flags & DT_NODE_OPEN) && total >= neighbourNode->total)
				continue;
			// The node is already visited and process, and the new result is worse, skip.
			if ((neighbourNode->flags & DT_NODE_CLOSED) && total >= neighbourNode->total)
				continue;
			// Add or update the node.
			neighbourNode->pidx = m_nodePool->getNodeIdx(bestNode);
			neighbourNode->id = neighbourRef;
			neighbourNode->flags = (neighbourNode->flags & ~DT_NODE_CLOSED);
			neighbourNode->cost = cost;
			neighbourNode->total = total;
			
			if (neighbourNode->flags & DT_NODE_OPEN) {
				// Already in open, update node location.
				m_openList->modify(neighbourNode);  // total值变小了，openlist是优先队列，节点可能需要上升
			} else {
				// Put the node in open list.
				neighbourNode->flags |= DT_NODE_OPEN;
				m_openList->push(neighbourNode);
			}
			
			// Update nearest node to target so far.
			if (heuristic < lastBestNodeCost) {
				lastBestNodeCost = heuristic;
				lastBestNode = neighbourNode;
			}
		}
	}
	if (lastBestNode->id != endRef)
		status |= DT_PARTIAL_RESULT;  // 没有到达终点，只得到部分结果
	
	// Reverse the path.
	dtNode* prev = 0;
	dtNode* node = lastBestNode;
	do {
		dtNode* next = m_nodePool->getNodeAtIdx(node->pidx);
		node->pidx = m_nodePool->getNodeIdx(prev);
		prev = node;
		node = next;
	} while (node);
	
	// Store path
	node = prev;
	int n = 0;
	do {
		path[n++] = node->id;
		if (n >= maxPath) {
			status |= DT_BUFFER_TOO_SMALL;
			break;
		}
		node = m_nodePool->getNodeAtIdx(node->pidx);
	} while (node);
	
	*pathCount = n;
	return status;
}
```