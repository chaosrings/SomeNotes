# NGUI渲染

核心是三个组件 UIPanel,UIWidget,UIDrawCall.

## 深度管理

UIPanel中有一个包含全局所有active Panel的List<UIPanel>,其中的Panel根据深度排序:

```csharp
UIPanel.cs
/// <summary>
/// Function that can be used to depth-sort panels.
/// </summary>
static public int CompareFunc (UIPanel a, UIPanel b)
{
	if (a != b && a != null && b != null)
	{
		if (a.mDepth < b.mDepth) return -1;
		if (a.mDepth > b.mDepth) return 1;
		return (a.GetInstanceID() < b.GetInstanceID()) ? -1 : 1;
	}
	return 0;
}
```

每个UIPanel管理所有子节点的UIWidget,UIWidget也会根据Depth进行排序:

```csharp

/// <summary>
/// Static widget comparison function used for inter-panel depth sorting.
/// </summary>
static public int PanelCompareFunc (UIWidget left, UIWidget right)
{
	if (left.mDepth < right.mDepth) return -1;
	if (left.mDepth > right.mDepth) return 1;

	Material leftMat = left.material;
	Material rightMat = right.material;

	if (leftMat == rightMat) return 0;
	if (leftMat != null) return -1;
	if (rightMat != null) return 1;

    //It seems that, logic will never go here ??? OK I will protect it
    int leftInsId = leftMat != null ? leftMat.GetInstanceID() : 0;
    int rightInsId = rightMat != null ? rightMat.GetInstanceID() : 0;

    return (leftInsId < rightInsId) ? -1 : 1;
}
```

在UIPanel.FillAllDrawCalls中会遍历所有的Widgets,生成UIDrawCall,前后两个Widget的Material,Texture,Shader相同则会进行合批.

UIDrawCall会生成顶点法线uv材质,贴图等所有渲染需要的元素,提交到GPU进行绘制:

```csharp

public class UIDrawCall : MonoBehaviour
{
    Material		mMaterial;		// Material used by this draw call
    Texture			mTexture;		// Main texture used by the material
    Shader			mShader;		// Shader used by the dynamically created material
    int				mClipCount = 0;	// Number of times the draw call's content is getting clipped
    Transform		mTrans;			// Cached transform
    Mesh			mMesh;			// First generated mesh
    MeshFilter		mFilter;		// Mesh filter for this draw call
    MeshRenderer	mRenderer;		// Mesh renderer for this screen
    Material		mDynamicMat;	// Instantiated material
    int[]			mIndices;		// Cached indices
    [HideInInspector][System.NonSerialized] public int widgetCount = 0;
    [HideInInspector][System.NonSerialized] public int depthStart = int.MaxValue;
    [HideInInspector][System.NonSerialized] public int depthEnd = int.MinValue;
    [HideInInspector][System.NonSerialized] public UIPanel manager;
    [HideInInspector][System.NonSerialized] public UIPanel panel;
    [HideInInspector][System.NonSerialized] public Texture2D clipTexture;
    [HideInInspector][System.NonSerialized] public bool alwaysOnScreen = false;
    [HideInInspector][System.NonSerialized] public BetterList<Vector3> verts = new BetterList<Vector3>();
    [HideInInspector][System.NonSerialized] public BetterList<Vector3> norms = new BetterList<Vector3>();
    [HideInInspector][System.NonSerialized] public BetterList<Vector4> tans = new BetterList<Vector4>();
    [HideInInspector][System.NonSerialized] public BetterList<Vector2> uvs = new BetterList<Vector2>();
    [HideInInspector][System.NonSerialized] public BetterList<Color32> cols = new BetterList<Color32>();
}
```

前面提到的UIPanel Depth,UIWidget Depth,最终生效的原理都是影响了UIDrawCall的渲染属性(材质的RenderQueue,sortingOrder):

```csharp

public class UIDrawCall : MonoBehaviour
{
    public int renderQueue
	{
		get
		{
			return mRenderQueue;
		}
		set
		{
			if (mRenderQueue != value)
			{
				mRenderQueue = value;
				if (mDynamicMat != null)
				{
                    if (mShareMatGroup == UIPanel.ShareMatGroup.E_None)
                    {
                        mDynamicMat.renderQueue = value;
                    }
                    else
                    {
                        mRebuildMat = true;
                    }
				}
			}
		}
	}
}

```

在UIPanel的默认自动模式Automatic下,UIPanel生成的所有DrawCall的renderQueue数值是startingRenderQueue+index:

```csharp
//UIPanel.cs

void UpdateDrawCalls ()
{
    for (int i = 0; i < drawCalls.Count; ++i)
    {
        UIDrawCall dc = drawCalls[i];
        ...
        dc.renderQueue = (renderQueue == RenderQueue.Explicit) ? startingRenderQueue : startingRenderQueue + i;
    }
}

```

而每个Panel的startingRenderQueue都会根据上一个Panel的DrawCall次数动态计算:

curPanel.startingRenderQueue = lastPanel.startingRenderQueue + lastPanel.drawCalls.Count

```csharp
//UIPanel.cs

void LateUpdate ()
{
    ...
    // Update all draw calls, making them draw in the right order
    for (int i = 0, imax = list.Count; i < imax; ++i)
    {
        UIPanel p = list[i];
        if (p.renderQueue == RenderQueue.Automatic)
        {
			p.startingRenderQueue = rq;
            if (!dontUpdate)
            {
                p.UpdateDrawCalls();
            }
            int offset = 0; //可以提供一个额外的冗余数值,让粒子特效能够插进入
            rq += p.drawCalls.Count + offset;
        }
}
}
```
