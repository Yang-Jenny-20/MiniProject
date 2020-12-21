#include <windows.h>

#include <algorithm>

#include <cmath>

#include "tchar.h"

#include "arxHeaders.h"


using namespace std;

//辅助实体类
//包括一个圆，三条直线
class CMultiJigEntity : public AcDbEntity
{
public:

	CMultiJigEntity(const AcGePoint3d& centerPoint);
	~CMultiJigEntity();
	//接口
	void setRadius(double dRadius);

	void setLength(AcGePoint3d& centerPoint, double dRadius);

	void appendToCurrentSpace();
private:

	//实体组成部分用一个Entity数组表示
	AcArray<AcDbEntity*> m_Arr;

};

CMultiJigEntity::CMultiJigEntity(const AcGePoint3d& centerPoint)
{

	AcDbCircle* pCirc;

	//初始化圆
	double radius = 0.0001;

	pCirc = new AcDbCircle(centerPoint, AcGeVector3d::kZAxis, radius);

	m_Arr.append(pCirc);

	//画三条线
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
	//析构，释放内存
	for (int i = 0; i < m_Arr.length(); i++)
	{
		delete m_Arr[i];
	}
}

//update()调用的函数，通过输入更新圆的半径
inline void CMultiJigEntity::setRadius(double dRadius)
{

	AcDbCircle* pCirc = AcDbCircle::cast(m_Arr[0]);

	pCirc->setRadius(dRadius);
}

//update()调用的函数，通过输入的半径和圆心计算出三条直线的位置
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

//继承jig类
class CMultiJig : public AcEdJig
{
public:
	CMultiJig(const AcGePoint3d& centerPoint);

	void doIt();

	// 重写AcEdJig类的三个函数
	virtual DragStatus sampler();

	virtual Adesk::Boolean update();

	//返回一个实体的指针
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

//第一步采数据
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
		//无变化，不更新
		return AcEdJig::kNoChange;
	}

	return stat;

}

//更新
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
	//初始化一个实例
	m_pEnt = new CMultiJigEntity(m_CenterPoint);

	setDispPrompt(_T("/nPlease input the radius:"));

	//调用sampler，update，entity函数更新实例的各个属性
	if (drag() == AcEdJig::kNormal)
	{
		m_pEnt->appendToCurrentSpace();
	}

	delete m_pEnt;
}

//测试函数
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
	//调用Myjig函数
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
//加载entry
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
