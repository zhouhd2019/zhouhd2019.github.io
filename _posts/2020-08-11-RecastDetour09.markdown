---
layout:     post
title:      "Navmesh"
subtitle:   "Recast Detour 09"
date:       2020-08-11 11:45:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - PathFinding
---

这里是Recast构建寻路网格的最后一步，归纳一下之前的数据，创建Navmesh
```c++
unsigned char* navData = 0;
int navDataSize = 0;
if (m_cfg.maxVertsPerPoly <= DT_VERTS_PER_POLYGON)
{
	// Update poly flags from areas.
	// 转换标志位
	for (int i = 0; i < m_pmesh->npolys; ++i) {
		if (m_pmesh->areas[i] == RC_WALKABLE_AREA)
			m_pmesh->areas[i] = SAMPLE_POLYAREA_GROUND;
		
		if (m_pmesh->areas[i] == SAMPLE_POLYAREA_GROUND ||
			m_pmesh->areas[i] == SAMPLE_POLYAREA_GRASS ||
			m_pmesh->areas[i] == SAMPLE_POLYAREA_ROAD) {
			m_pmesh->flags[i] = SAMPLE_POLYFLAGS_WALK;
		}
		else if (m_pmesh->areas[i] == SAMPLE_POLYAREA_WATER) {
			m_pmesh->flags[i] = SAMPLE_POLYFLAGS_SWIM;
		}
		else if (m_pmesh->areas[i] == SAMPLE_POLYAREA_DOOR) {
			m_pmesh->flags[i] = SAMPLE_POLYFLAGS_WALK | SAMPLE_POLYFLAGS_DOOR;
		}
	}
	
	dtNavMeshCreateParams params;
	// 设置dtNavMeshCreateParams的参数
	
	if (!dtCreateNavMeshData(&params, &navData, &navDataSize)) {
		m_ctx->log(RC_LOG_ERROR, "Could not build Detour navmesh.");
		return 0;
	}
}
```
再看一下dtCreateNavMeshData，没有什么特殊逻辑，重新分配足够的对齐空间，存一下之前的数据即可
```c++
bool dtCreateNavMeshData(dtNavMeshCreateParams* params, unsigned char** outData, int* outDataSize)
{
	// 参数检查，省略......
	const int nvp = params->nvp;
	// Classify off-mesh connection points. We store only the connections
	// whose start point is inside the tile.
	unsigned char* offMeshConClass = 0;
	int storedOffMeshConCount = 0;
	int offMeshConLinkCount = 0;
	
	if (params->offMeshConCount > 0) {
		// 处理非网格连接(手动指定的连接路线)，省略......
	}
	// Off-mesh connectionss are stored as polygons, adjust values.
	const int totPolyCount = params->polyCount + storedOffMeshConCount;
	const int totVertCount = params->vertCount + storedOffMeshConCount*2;
	
	// Find portal edges which are at tile borders.
	int edgeCount = 0;
	int portalCount = 0;
	for (int i = 0; i < params->polyCount; ++i) {
		const unsigned short* p = &params->polys[i*2*nvp];
		for (int j = 0; j < nvp; ++j) {
			if (p[j] == MESH_NULL_IDX) break;
			edgeCount++;
			if (p[nvp+j] & 0x8000) {
				unsigned short dir = p[nvp+j] & 0xf;
				if (dir != 0xf)
					portalCount++;
			}
		}
	}

	const int maxLinkCount = edgeCount + portalCount*2 + offMeshConLinkCount*2;
	// Find unique detail vertices.
	int uniqueDetailVertCount = 0;
	int detailTriCount = 0;
	if (params->detailMeshes) {
		// Has detail mesh, count unique detail vertex count and use input detail tri count.
		detailTriCount = params->detailTriCount;
		for (int i = 0; i < params->polyCount; ++i) {
			const unsigned short* p = &params->polys[i*nvp*2];
			int ndv = params->detailMeshes[i*4+1];
			int nv = 0;
			for (int j = 0; j < nvp; ++j) {
				if (p[j] == MESH_NULL_IDX) break;
				nv++;
			}
			ndv -= nv;
			uniqueDetailVertCount += ndv;
		}
	} else {
		// No input detail mesh, build detail mesh from nav polys.
		uniqueDetailVertCount = 0; // No extra detail verts.
		detailTriCount = 0;
		for (int i = 0; i < params->polyCount; ++i) {
			const unsigned short* p = &params->polys[i*nvp*2];
			int nv = 0;
			for (int j = 0; j < nvp; ++j) {
				if (p[j] == MESH_NULL_IDX) break;
				nv++;
			}
			detailTriCount += nv-2;
		}
	}
	
	// Calculate data size
	const int headerSize = dtAlign4(sizeof(dtMeshHeader));
	const int vertsSize = dtAlign4(sizeof(float)*3*totVertCount);
	const int polysSize = dtAlign4(sizeof(dtPoly)*totPolyCount);
	const int linksSize = dtAlign4(sizeof(dtLink)*maxLinkCount);
	const int detailMeshesSize = dtAlign4(sizeof(dtPolyDetail)*params->polyCount);
	const int detailVertsSize = dtAlign4(sizeof(float)*3*uniqueDetailVertCount);
	const int detailTrisSize = dtAlign4(sizeof(unsigned char)*4*detailTriCount);
	const int bvTreeSize = params->buildBvTree ? dtAlign4(sizeof(dtBVNode)*params->polyCount*2) : 0;
	const int offMeshConsSize = dtAlign4(sizeof(dtOffMeshConnection)*storedOffMeshConCount);
	const int dataSize = headerSize + vertsSize + polysSize + linksSize +
						 detailMeshesSize + detailVertsSize + detailTrisSize +
						 bvTreeSize + offMeshConsSize;
	unsigned char* data = (unsigned char*)dtAlloc(sizeof(unsigned char)*dataSize, DT_ALLOC_PERM);
	memset(data, 0, dataSize);
	
	unsigned char* d = data;
	dtMeshHeader* header = (dtMeshHeader*)d; d += headerSize;
	float* navVerts = (float*)d; d += vertsSize;
	dtPoly* navPolys = (dtPoly*)d; d += polysSize;
	d += linksSize;
	dtPolyDetail* navDMeshes = (dtPolyDetail*)d; d += detailMeshesSize;
	float* navDVerts = (float*)d; d += detailVertsSize;
	unsigned char* navDTris = (unsigned char*)d; d += detailTrisSize;
	dtBVNode* navBvtree = (dtBVNode*)d; d += bvTreeSize;
	dtOffMeshConnection* offMeshCons = (dtOffMeshConnection*)d; d += offMeshConsSize;
	
	// Store header
	header->magic = DT_NAVMESH_MAGIC;
	// 设置header字段，省略......
	
	const int offMeshVertsBase = params->vertCount;
	const int offMeshPolyBase = params->polyCount;
	// Store vertices
	// Mesh vertices
	for (int i = 0; i < params->vertCount; ++i) {
		const unsigned short* iv = &params->verts[i*3];
		float* v = &navVerts[i*3];
		v[0] = params->bmin[0] + iv[0] * params->cs;
		v[1] = params->bmin[1] + iv[1] * params->ch;
		v[2] = params->bmin[2] + iv[2] * params->cs;
	}
	// Off-mesh link vertices.
	int n = 0;
	for (int i = 0; i < params->offMeshConCount; ++i) {
		// Only store connections which start from this tile.
		if (offMeshConClass[i*2+0] == 0xff) {
			const float* linkv = &params->offMeshConVerts[i*2*3];
			float* v = &navVerts[(offMeshVertsBase + n*2)*3];
			dtVcopy(&v[0], &linkv[0]);
			dtVcopy(&v[3], &linkv[3]);
			n++;
		}
	}
	// Store polygons
	// Mesh polys
	const unsigned short* src = params->polys;
	for (int i = 0; i < params->polyCount; ++i) {
		dtPoly* p = &navPolys[i];
		p->vertCount = 0;
		p->flags = params->polyFlags[i];
		p->setArea(params->polyAreas[i]);
		p->setType(DT_POLYTYPE_GROUND);
		for (int j = 0; j < nvp; ++j) {
			if (src[j] == MESH_NULL_IDX) break;
			p->verts[j] = src[j];
			if (src[nvp+j] & 0x8000) {
				// Border or portal edge.
				unsigned short dir = src[nvp+j] & 0xf;
				if (dir == 0xf) // Border
					p->neis[j] = 0;
				else if (dir == 0) // Portal x-
					p->neis[j] = DT_EXT_LINK | 4;
				else if (dir == 1) // Portal z+
					p->neis[j] = DT_EXT_LINK | 2;
				else if (dir == 2) // Portal x+
					p->neis[j] = DT_EXT_LINK | 0;
				else if (dir == 3) // Portal z-
					p->neis[j] = DT_EXT_LINK | 6;
			} else {
				// Normal connection
				p->neis[j] = src[nvp+j]+1;
			}
			p->vertCount++;
		}
		src += nvp*2;
	}
	// Off-mesh connection vertices.
	// 省略......

	// Store detail meshes and vertices.
	// The nav polygon vertices are stored as the first vertices on each mesh.
	// We compress the mesh data by skipping them and using the navmesh coordinates.
	if (params->detailMeshes) {
		unsigned short vbase = 0;
		for (int i = 0; i < params->polyCount; ++i) {
			dtPolyDetail& dtl = navDMeshes[i];
			const int vb = (int)params->detailMeshes[i*4+0];
			const int ndv = (int)params->detailMeshes[i*4+1];
			const int nv = navPolys[i].vertCount;
			dtl.vertBase = (unsigned int)vbase;
			dtl.vertCount = (unsigned char)(ndv-nv);
			dtl.triBase = (unsigned int)params->detailMeshes[i*4+2];
			dtl.triCount = (unsigned char)params->detailMeshes[i*4+3];
			// Copy vertices except the first 'nv' verts which are equal to nav poly verts.
			if (ndv-nv) {
				memcpy(&navDVerts[vbase*3], &params->detailVerts[(vb+nv)*3], sizeof(float)*3*(ndv-nv));
				vbase += (unsigned short)(ndv-nv);
			}
		}
		// Store triangles.
		memcpy(navDTris, params->detailTris, sizeof(unsigned char)*4*params->detailTriCount);
	}
	else {
		// Create dummy detail mesh by triangulating polys.
		// 没有detail mesh，需要另外创建，省略......
	}

	// Store and create BVtree.
	// TODO: take detail mesh into account! use byte per bbox extent?
	if (params->buildBvTree) {
		createBVTree(params->verts, params->vertCount, params->polys, params->polyCount,
					 nvp, params->cs, params->ch, params->polyCount*2, navBvtree);
	}
	// Store Off-Mesh connections.
	// off-mesh，省略......
	*outData = data;
	*outDataSize = dataSize;
	return true;
}
```
稍微特别的地方是createBVTree，逐个计算多边形的包围盒，然后根据包围盒长宽高，每次沿最大分割轴(x/y/z)来分割包围盒