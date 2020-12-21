#include <windows.h>

#include <algorithm>

#include "tchar.h"

#include "arxHeaders.h"

#include <algorithm>
	
#include <cmath>

using namespace std;

class CMultiJigEntity : public AcDbEntity
{

public:

	CMultiJigEntity(const AcGePoint3d& centerPoint);
	
	~CMultiJigEntity();
	
	void setRadius(double dRadius);

	void setLength(AcGePoint3d& centerPoint, double dRadius);

	void appendToCurrentSpace();
private:

	AcArray<AcDbEntity*> m_Arr;

};

CMultiJigEntity::CMultiJigEntity(const AcGePoint3d& centerPoint)
{

	AcDbCircle* pCirc;
	
	double radius = 0.0001;

	pCirc = new AcDbCircle(centerPoint, AcGeVector3d::kZAxis, radius);

	m_Arr.append(pCirc);

	AcGePoint3d a = centerPoint;

	AcGePoint3d b = AcGePoint3d(centerPoint[0], centerPoint[1] + radius, 0.0);

	AcDbLine* pLine = new AcDbLine(a, b);

	m_Arr.append(pLine);

	b = AcGePoint3d(centerPoint[0] - radius * 0.2 * pow(3, 0.5), centerPoint[1] - 0.5 * radius, 0.0);

	pLine = new AcDbLine(a, b);

	m_Arr.append(pLine);

	b = AcGePoint3d(centerPoint[0] + radius * 0.5, centerPoint[1] - centerPoint[1] - 0.5 * radius, 0.0);

	pLine = new AcDbLine(a, b);

	m_Arr.append(pLine);
}

CMultiJigEntity::~CMultiJigEntity()
{
	for (int i = 0; i < m_Arr.length(); i++)
	{
		delete m_Arr[i];
	}
}

inline void CMultiJigEntity::setRadius(double dRadius)
{

	AcDbCircle* pCirc = AcDbCircle::cast(m_Arr[0]);

	pCirc->setRadius(dRadius);
}

inline void CMultiJigEntity::setLength(AcGePoint3d& centerPoint, double dRadius)
{
	AcGePoint3d a = centerPoint;

	AcGePoint3d b = AcGePoint3d(centerPoint[0], centerPoint[1] + dRadius, centerPoint[2]);

	AcDbLine* pLine = AcDbLine::cast(m_Arr[1]);

	pLine->setEndPoint(AcGePoint3d(centerPoint[0], centerPoint[1] + dRadius, centerPoint[2]));

	pLine = AcDbLine::cast(m_Arr[2]);

	pLine->setEndPoint(AcGePoint3d(centerPoint[0] - dRadius * 0.5 * pow(3, 0.5), centerPoint[1] - 0.5 * dRadius, centerPoint[2]));

	pLine = AcDbLine::cast(m_Arr[3]);

	pLine->setEndPoint(AcGePoint3d(centerPoint[0] + dRadius * 0.5 * pow(3, 0.5), centerPoint[1] - 0.5 * dRadius, centerPoint[2]));

}

void CMultiJigEntity::appendToCurrentSpace()
{
	AcDbDatabase* pDb = acdbCurDwg();

	AcDbBlockTable* pBlockTable;

	pDb->getBlockTable(pBlockTable, AcDb::kForRead);

	AcDbBlockTableRecord* pBlkRec;

	if (pDb->tilemode())
	{
		pBlockTable->getAt(ACDB_MODEL_SPACE, pBlkRec, AcDb::kForWrite);
	}
	else
	{
		pBlockTable->getAt(ACDB_PAPER_SPACE, pBlkRec, AcDb::kForWrite);
	}

	pBlockTable->close();

	for (int i = 0; i < m_Arr.length(); i++)
	{
		if (Acad::eOk == pBlkRec->appendAcDbEntity(m_Arr[i]))
		{
			m_Arr[i]->setDatabaseDefaults();

			m_Arr[i]->close();
		}
		else
		{
			delete m_Arr[i];
		}
	}
	pBlkRec->close();

	m_Arr.removeAll();
}

class CMultiJig : public AcEdJig
{
public:
	CMultiJig(const AcGePoint3d& centerPoint);

	void doIt();

	virtual DragStatus sampler();

	virtual Adesk::Boolean update();

	virtual AcDbEntity* entity() const;
	
private:

	CMultiJigEntity* m_pEnt;
	AcGePoint3d				m_CenterPoint;
	double					m_dRadius;
	unsigned int			m_NumCircles;

};

CMultiJig::CMultiJig(const AcGePoint3d& centerPoint) : m_CenterPoint(centerPoint)
{
}

AcEdJig::DragStatus CMultiJig::sampler()
{
	static double dTempRadius;

	DragStatus stat = acquireDist(m_dRadius, m_CenterPoint);

	if (dTempRadius != m_dRadius)
	{
		dTempRadius = m_dRadius;
	}
	else if (stat == AcEdJig::kNormal)
	{
		
		return AcEdJig::kNoChange;
	}

	return stat;

}

Adesk::Boolean  CMultiJig::update()
{
	m_pEnt->setRadius(m_dRadius);

	m_pEnt->setLength(m_CenterPoint, m_dRadius);

	return Adesk::kTrue;
}

AcDbEntity* CMultiJig::entity() const
{
	return m_pEnt;
}

void CMultiJig::doIt()
{
	
	m_pEnt = new CMultiJigEntity(m_CenterPoint);

	setDispPrompt(_T("/nPlease input the radius:"));

	if (drag() == AcEdJig::kNormal)
	{
		m_pEnt->appendToCurrentSpace();
	}

	delete m_pEnt;
}

static void MyJig(void)
{
	AcGePoint3d centerPoint;

	if (RTNORM == acedGetPoint(NULL, _T("/nPlease input the center point:"), asDblArray(centerPoint)))
	{

		CMultiJig* pJig = new CMultiJig(centerPoint);

		pJig->doIt();

		delete pJig;
	}
}

void
initApp()
{
	
	acedRegCmds->addCommand(_T("ASDK_VISUAL_ELLIPSE"),
		_T("ASDK_BenzLogo"), _T("BenzLogo"), ACRX_CMD_MODAL,
		MyJig);

}

void
unloadApp()
{

	acedRegCmds->removeGroup(_T("ASDK_BenzLogo"));

}

extern "C" AcRx::AppRetCode
acrxEntryPoint(AcRx::AppMsgCode msg, void* appId)
{
	switch (msg) {

	case AcRx::kInitAppMsg:

		acrxDynamicLinker->unlockApplication(appId);

		acrxDynamicLinker->registerAppMDIAware(appId);

		initApp();

		break;

	case AcRx::kUnloadAppMsg:

		unloadApp();

	}

	return AcRx::kRetOK;

}
