```react
import React, { useState, useEffect, useRef, useCallback } from 'react';
import { Delete, Trash2, ZoomIn, ZoomOut, Calculator, Activity, Maximize, RefreshCcw, ChevronDown, ChevronUp } from 'lucide-react';

// KaTeX渲染组件
const MathDisplay = ({ tex, isError }) => {
  const containerRef = useRef(null);

  useEffect(() => {
    if (window.katex && containerRef.current && tex && !isError) {
      try {
        window.katex.render(tex, containerRef.current, {
          throwOnError: false,
          displayMode: true,
        });
      } catch (e) {
        containerRef.current.innerText = tex;
      }
    } else if (containerRef.current) {
      containerRef.current.innerText = isError ? "无效表达式" : tex;
    }
  }, [tex, isError]);

  return (
    <div
      ref={containerRef}
      className={`text-2xl font-serif flex items-center justify-center min-h-[4rem] overflow-x-auto whitespace-nowrap p-2 ${
        isError ? 'text-red-500 text-lg' : 'text-slate-800'
      }`}
    />
  );
};

export default function App() {
  const [isReady, setIsReady] = useState(false);
  const [expression, setExpression] = useState('');
  const [result, setResult] = useState('');
  const [isError, setIsError] = useState(false);
  const [latex, setLatex] = useState('');
  
  // 绘图相关状态
  const [functions, setFunctions] = useState([]);
  const [offsetX, setOffsetX] = useState(0);
  const [offsetY, setOffsetY] = useState(0);
  const [scale, setScale] = useState(40); // px per unit
  const [mobileTab, setMobileTab] = useState('calc'); // 'calc' or 'graph'
  const [showAdvanced, setShowAdvanced] = useState(true);

  const canvasContainerRef = useRef(null);
  const canvasRef = useRef(null);
  const [dimensions, setDimensions] = useState({ w: 0, h: 0 });
  const [isDragging, setIsDragging] = useState(false);
  const [dragStart, setDragStart] = useState({ x: 0, y: 0 });

  const COLORS = ['#3b82f6', '#ef4444', '#10b981', '#f59e0b', '#8b5cf6'];

  // 动态加载外部依赖 (Math.js 和 KaTeX)
  useEffect(() => {
    const loadScript = (src) => new Promise((resolve, reject) => {
      const s = document.createElement('script');
      s.src = src;
      s.onload = resolve;
      s.onerror = reject;
      document.head.appendChild(s);
    });

    const loadCSS = (href) => {
      const l = document.createElement('link');
      l.rel = 'stylesheet';
      l.href = href;
      document.head.appendChild(l);
    };

    loadCSS("https://cdn.jsdelivr.net/npm/katex@0.16.8/dist/katex.min.css");

    Promise.all([
      loadScript("https://cdnjs.cloudflare.com/ajax/libs/mathjs/11.8.0/math.min.js"),
      loadScript("https://cdn.jsdelivr.net/npm/katex@0.16.8/dist/katex.min.js")
    ]).then(() => setIsReady(true)).catch(console.error);
  }, []);

  // 监听输入并更新LaTeX和结果
  useEffect(() => {
    if (!isReady || !window.math) return;
    
    if (!expression) {
      setLatex('');
      setResult('');
      setIsError(false);
      return;
    }

    try {
      const node = window.math.parse(expression);
      setLatex(node.toTex());
      setIsError(false);

      try {
        const res = window.math.evaluate(expression);
        if (typeof res === 'number') {
          setResult('= ' + window.math.format(res, { precision: 10 }));
        } else {
          setResult('= ' + res.toString());
        }
      } catch (evalErr) {
        if (evalErr.message.includes('Undefined symbol')) {
          setResult('包含变量的函数表达式');
        } else {
          setResult('');
        }
      }
    } catch (parseErr) {
      setLatex(expression); // Fallback
      setIsError(true);
      setResult('');
    }
  }, [expression, isReady]);

  // 响应式画布尺寸
  useEffect(() => {
    if (!canvasContainerRef.current) return;
    const observer = new ResizeObserver(entries => {
      for (let entry of entries) {
        setDimensions({ w: entry.contentRect.width, h: entry.contentRect.height });
      }
    });
    observer.observe(canvasContainerRef.current);
    return () => observer.disconnect();
  }, []);

  // 绘制引擎
  const draw = useCallback(() => {
    if (!canvasRef.current || dimensions.w === 0 || !window.math) return;
    
    const canvas = canvasRef.current;
    const ctx = canvas.getContext('2d');
    const dpr = window.devicePixelRatio || 1;
    const w = dimensions.w;
    const h = dimensions.h;

    canvas.width = w * dpr;
    canvas.height = h * dpr;
    canvas.style.width = `${w}px`;
    canvas.style.height = `${h}px`;
    ctx.scale(dpr, dpr);

    ctx.clearRect(0, 0, w, h);

    // 坐标转换函数
    const toCanvasX = (mx) => (mx - offsetX) * scale + w / 2;
    const toCanvasY = (my) => -(my - offsetY) * scale + h / 2;
    const toMathX = (cx) => (cx - w / 2) / scale + offsetX;
    const toMathY = (cy) => -(cy - h / 2) / scale + offsetY;

    // 绘制网格
    ctx.strokeStyle = '#e2e8f0';
    ctx.lineWidth = 1;
    let step = 1;
    if (scale < 15) step = 5;
    if (scale < 5) step = 10;
    if (scale > 80) step = 0.5;

    const startX = Math.floor(toMathX(0) / step) * step;
    const endX = Math.ceil(toMathX(w) / step) * step;
    for (let x = startX; x <= endX; x += step) {
      const cx = toCanvasX(x);
      ctx.beginPath(); ctx.moveTo(cx, 0); ctx.lineTo(cx, h); ctx.stroke();
      if (x !== 0) {
        ctx.fillStyle = '#94a3b8';
        ctx.font = '10px sans-serif';
        ctx.fillText(Number(x.toPrecision(4)), cx + 4, toCanvasY(0) + 12);
      }
    }

    const startY = Math.floor(toMathY(h) / step) * step;
    const endY = Math.ceil(toMathY(0) / step) * step;
    for (let y = startY; y <= endY; y += step) {
      const cy = toCanvasY(y);
      ctx.beginPath(); ctx.moveTo(0, cy); ctx.lineTo(w, cy); ctx.stroke();
      if (y !== 0) {
        ctx.fillStyle = '#94a3b8';
        ctx.fillText(Number(y.toPrecision(4)), toCanvasX(0) + 4, cy - 4);
      }
    }

    // 绘制坐标轴
    ctx.strokeStyle = '#475569';
    ctx.lineWidth = 2;
    const y0 = toCanvasY(0);
    if (y0 >= 0 && y0 <= h) {
      ctx.beginPath(); ctx.moveTo(0, y0); ctx.lineTo(w, y0); ctx.stroke();
    }
    const x0 = toCanvasX(0);
    if (x0 >= 0 && x0 <= w) {
      ctx.beginPath(); ctx.moveTo(x0, 0); ctx.lineTo(x0, h); ctx.stroke();
    }

    // 绘制所有函数
    functions.forEach(func => {
      try {
        const node = window.math.parse(func.expr);
        const code = node.compile();

        ctx.beginPath();
        ctx.strokeStyle = func.color;
        ctx.lineWidth = 2;
        ctx.lineJoin = 'round';

        let first = true;
        for (let cx = 0; cx <= w; cx += 2) {
          const mx = toMathX(cx);
          try {
            const my = code.evaluate({ x: mx, e: Math.E, pi: Math.PI });
            if (typeof my === 'number' && !isNaN(my)) {
              const cy = toCanvasY(my);
              // 防止跨越渐近线绘制（例如tan(x)）
              if (cy < -2000 || cy > h + 2000) {
                first = true;
                continue;
              }
              if (first) { ctx.moveTo(cx, cy); first = false; }
              else { ctx.lineTo(cx, cy); }
            } else {
              first = true;
            }
          } catch (e) {
            first = true;
          }
        }
        ctx.stroke();
      } catch (e) { }
    });
  }, [dimensions, offsetX, offsetY, scale, functions]);

  useEffect(() => {
    draw();
  }, [draw]);

  // 画布交互事件
  const handleWheel = (e) => {
    e.preventDefault();
    const zoomFactor = e.deltaY > 0 ? 0.9 : 1.1;
    setScale(prev => Math.max(1, Math.min(prev * zoomFactor, 500)));
  };

  const handlePointerDown = (e) => {
    setIsDragging(true);
    setDragStart({ x: e.clientX, y: e.clientY });
  };

  const handlePointerMove = (e) => {
    if (!isDragging) return;
    const dx = e.clientX - dragStart.x;
    const dy = e.clientY - dragStart.y;
    setOffsetX(prev => prev - dx / scale);
    setOffsetY(prev => prev + dy / scale);
    setDragStart({ x: e.clientX, y: e.clientY });
  };

  const handlePointerUp = () => setIsDragging(false);

  // 键盘与操作逻辑
  const handleKey = (key) => {
    if (key === 'C') {
      setExpression('');
      setResult('');
    } else if (key === 'DEL') {
      setExpression(prev => prev.slice(0, -1));
    } else if (key === '=') {
      // 触发强制计算
    } else if (key === 'Plot') {
      if (expression && !isError) {
        setFunctions(prev => {
          // 避免重复绘制
          if (prev.find(f => f.expr === expression)) return prev;
          const color = COLORS[prev.length % COLORS.length];
          return [...prev, { id: Date.now(), expr: expression, color }];
        });
        if (window.innerWidth < 768) setMobileTab('graph');
      }
    } else if (['sin', 'cos', 'tan', 'asin', 'acos', 'atan', 'sinh', 'cosh', 'tanh', 'sqrt', 'log', 'ln', 'abs'].includes(key)) {
      if (key === 'ln') {
        setExpression(prev => prev + 'log('); // Math.js 使用 log(x) 作为自然对数
      } else if (key === 'log') {
        setExpression(prev => prev + 'log10('); // Math.js 使用 log10(x) 作为以10为底
      } else {
        setExpression(prev => prev + key + '(');
      }
    } else if (key === '!') {
      setExpression(prev => prev + '!');
    } else if (key === 'x^2') {
      setExpression(prev => prev + '^2');
    } else if (key === 'x^y') {
      setExpression(prev => prev + '^');
    } else {
      setExpression(prev => prev + key);
    }
  };

  const removeFunction = (id) => {
    setFunctions(prev => prev.filter(f => f.id !== id));
  };

  const resetView = () => {
    setOffsetX(0);
    setOffsetY(0);
    setScale(40);
  };

  const btnClass = "flex items-center justify-center w-full h-full min-h-[2.5rem] sm:min-h-[2.75rem] md:min-h-[3rem] rounded-lg sm:rounded-xl text-sm sm:text-base font-medium transition-all active:scale-95 touch-manipulation";
  const numClass = `${btnClass} bg-white text-slate-800 shadow-sm border border-slate-100 hover:bg-slate-50`;
  const opClass = `${btnClass} bg-indigo-50 text-indigo-600 shadow-sm border border-indigo-100 hover:bg-indigo-100`;
  const funcClass = `${btnClass} bg-slate-100 text-slate-700 shadow-sm hover:bg-slate-200 text-[13px] sm:text-sm`;
  const actionClass = `${btnClass} bg-indigo-500 text-white shadow-md hover:bg-indigo-600`;

  if (!isReady) return (
    <div className="min-h-screen flex items-center justify-center bg-slate-50">
      <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-indigo-500"></div>
    </div>
  );

  return (
    <div className="flex flex-col md:flex-row h-screen w-full bg-slate-50 overflow-hidden font-sans">
      
      {/* 移动端标签页切换 */}
      <div className="md:hidden flex bg-white border-b border-slate-200 shrink-0">
        <button 
          onClick={() => setMobileTab('calc')}
          className={`flex-1 flex items-center justify-center gap-2 py-3 font-medium transition-colors ${mobileTab === 'calc' ? 'text-indigo-600 border-b-2 border-indigo-600' : 'text-slate-500'}`}
        >
          <Calculator size={18} /> 计算
        </button>
        <button 
          onClick={() => setMobileTab('graph')}
          className={`flex-1 flex items-center justify-center gap-2 py-3 font-medium transition-colors ${mobileTab === 'graph' ? 'text-indigo-600 border-b-2 border-indigo-600' : 'text-slate-500'}`}
        >
          <Activity size={18} /> 图表 {functions.length > 0 && <span className="bg-indigo-100 text-indigo-600 text-xs px-2 py-0.5 rounded-full">{functions.length}</span>}
        </button>
      </div>

      {/* 左侧计算面板 */}
      <div className={`w-full md:w-[420px] flex-col border-r border-slate-200 bg-white shadow-[2px_0_10px_rgba(0,0,0,0.02)] z-10 shrink-0 ${mobileTab === 'calc' ? 'flex h-[calc(100vh-50px)] md:h-screen' : 'hidden md:flex h-screen'}`}>
        
        {/* 显示区域 */}
        <div className="flex-none p-6 border-b border-slate-100 bg-slate-50/50 flex flex-col justify-end min-h-[160px]">
          <div className="text-slate-400 text-sm mb-2 text-right h-6">
            {expression || '等待输入...'}
          </div>
          <div className="flex-1 w-full flex items-center justify-end overflow-x-auto">
             <MathDisplay tex={latex} isError={isError} />
          </div>
          <div className="text-indigo-600 font-semibold text-xl text-right h-8 mt-2">
            {result}
          </div>
        </div>

        {/* 绘图函数列表（如果在计算器页也有空间则展示） */}
        {functions.length > 0 && (
          <div className="px-4 py-3 flex gap-2 overflow-x-auto border-b border-slate-100 bg-white items-center shrink-0 hide-scrollbar">
            <span className="text-xs font-semibold text-slate-400 uppercase tracking-wider">已绘图</span>
            {functions.map(f => (
              <div key={f.id} className="flex items-center gap-1.5 px-3 py-1.5 rounded-full bg-slate-50 border border-slate-200 shadow-sm shrink-0">
                <div className="w-2.5 h-2.5 rounded-full" style={{ backgroundColor: f.color }}></div>
                <span className="text-sm font-medium">{f.expr}</span>
                <button onClick={() => removeFunction(f.id)} className="text-slate-400 hover:text-red-500 transition-colors p-0.5">
                  <Trash2 size={14} />
                </button>
              </div>
            ))}
          </div>
        )}

        {/* 高级函数折叠开关 */}
        <div className="flex justify-between items-center px-4 py-2 bg-slate-50/80 border-b border-slate-100 shrink-0">
          <span className="text-xs font-semibold text-slate-400 tracking-wider">数学键盘</span>
          <button 
            onClick={() => setShowAdvanced(!showAdvanced)} 
            className="text-xs text-indigo-600 flex items-center gap-1 hover:text-indigo-700 transition-colors bg-indigo-50 px-2 py-1 rounded-md"
          >
            {showAdvanced ? <><ChevronDown size={14}/> 收起高级函数</> : <><ChevronUp size={14}/> 展开高级函数</>}
          </button>
        </div>

        {/* 键盘区域 */}
        <div className="p-2 sm:p-4 grid grid-cols-5 gap-1.5 sm:gap-2.5 mt-auto flex-1 bg-white overflow-y-auto hide-scrollbar">
          
          {showAdvanced && (
            <>
              <button onClick={() => handleKey('asin')} className={funcClass}>asin</button>
              <button onClick={() => handleKey('acos')} className={funcClass}>acos</button>
              <button onClick={() => handleKey('atan')} className={funcClass}>atan</button>
              <button onClick={() => handleKey('log')} className={funcClass}>log</button>
              <button onClick={() => handleKey('ln')} className={funcClass}>ln</button>
              
              <button onClick={() => handleKey('sinh')} className={funcClass}>sinh</button>
              <button onClick={() => handleKey('cosh')} className={funcClass}>cosh</button>
              <button onClick={() => handleKey('tanh')} className={funcClass}>tanh</button>
              <button onClick={() => handleKey('abs')} className={funcClass}>|x|</button>
              <button onClick={() => handleKey('!')} className={funcClass}>n!</button>
            </>
          )}

          <button onClick={() => handleKey('sin')} className={funcClass}>sin</button>
          <button onClick={() => handleKey('cos')} className={funcClass}>cos</button>
          <button onClick={() => handleKey('tan')} className={funcClass}>tan</button>
          <button onClick={() => handleKey('pi')} className={funcClass}>π</button>
          <button onClick={() => handleKey('C')} className={`${funcClass} text-red-500 font-bold bg-red-50 hover:bg-red-100`}>C</button>
          
          <button onClick={() => handleKey('sqrt')} className={funcClass}>√</button>
          <button onClick={() => handleKey('^')} className={funcClass}>x^y</button>
          <button onClick={() => handleKey('x')} className={`${funcClass} font-bold text-indigo-600 bg-indigo-50`}>x</button>
          <button onClick={() => handleKey('e')} className={funcClass}>e</button>
          <button onClick={() => handleKey('DEL')} className={`${funcClass} text-slate-500`}><Delete size={18}/></button>
          
          <button onClick={() => handleKey('(')} className={funcClass}>(</button>
          <button onClick={() => handleKey(')')} className={funcClass}>)</button>
          <button onClick={() => handleKey('/')} className={opClass}>÷</button>
          <button onClick={() => handleKey('*')} className={opClass}>×</button>
          <button onClick={() => handleKey('-')} className={opClass}>−</button>
          
          <button onClick={() => handleKey('7')} className={numClass}>7</button>
          <button onClick={() => handleKey('8')} className={numClass}>8</button>
          <button onClick={() => handleKey('9')} className={numClass}>9</button>
          <button onClick={() => handleKey('+')} className={`${opClass} row-span-2`}>+</button>
          <button onClick={() => handleKey('Plot')} className={`${actionClass} row-span-2 flex flex-col gap-1 shadow-indigo-200`}>
            <Activity size={18} /> <span className="text-[11px] sm:text-sm">绘制</span>
          </button>
          
          <button onClick={() => handleKey('4')} className={numClass}>4</button>
          <button onClick={() => handleKey('5')} className={numClass}>5</button>
          <button onClick={() => handleKey('6')} className={numClass}>6</button>
          
          <button onClick={() => handleKey('1')} className={numClass}>1</button>
          <button onClick={() => handleKey('2')} className={numClass}>2</button>
          <button onClick={() => handleKey('3')} className={numClass}>3</button>
          <button onClick={() => handleKey('=')} className={`${opClass} row-span-2 font-bold text-xl sm:text-2xl`}>=</button>
          <button onClick={() => setFunctions([])} className={`${funcClass} row-span-2 text-slate-500 flex flex-col gap-1`}>
            <RefreshCcw size={16}/> <span className="text-[10px] sm:text-xs">清空</span>
          </button>
          
          <button onClick={() => handleKey('0')} className={`${numClass} col-span-2`}>0</button>
          <button onClick={() => handleKey('.')} className={numClass}>.</button>
        </div>
      </div>

      {/* 右侧绘图面板 */}
      <div className={`flex-1 relative bg-slate-50 ${mobileTab === 'graph' ? 'flex h-[calc(100vh-50px)] md:h-screen' : 'hidden md:flex h-screen'}`}>
        {/* 画布容器 */}
        <div 
          ref={canvasContainerRef} 
          className="w-full h-full cursor-grab active:cursor-grabbing touch-none"
          onWheel={handleWheel}
          onPointerDown={handlePointerDown}
          onPointerMove={handlePointerMove}
          onPointerUp={handlePointerUp}
          onPointerLeave={handlePointerUp}
        >
          <canvas ref={canvasRef} className="block w-full h-full" />
        </div>

        {/* 绘图控件悬浮窗 */}
        <div className="absolute top-4 right-4 flex flex-col gap-2 bg-white/90 backdrop-blur-sm p-2 rounded-xl shadow-sm border border-slate-200">
          <button onClick={() => setScale(s => Math.min(s * 1.2, 500))} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="放大">
            <ZoomIn size={20} />
          </button>
          <button onClick={() => setScale(s => Math.max(s * 0.8, 1))} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="缩小">
            <ZoomOut size={20} />
          </button>
          <div className="w-full h-px bg-slate-200 my-1"></div>
          <button onClick={resetView} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="重置视图">
            <Maximize size={20} />
          </button>
        </div>

        {functions.length === 0 && (
          <div className="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 text-slate-400 flex flex-col items-center gap-3 pointer-events-none text-center px-4">
            <Activity size={48} className="text-slate-300" opacity={0.5} />
            <p className="text-lg font-medium">还没有绘制任何函数</p>
            <p className="text-sm">在左侧输入包含变量 <span className="font-mono bg-slate-200 px-1.5 py-0.5 rounded text-slate-600">x</span> 的表达式并点击"```react
import React, { useState, useEffect, useRef, useCallback } from 'react';
import { Delete, Trash2, ZoomIn, ZoomOut, Calculator, Activity, Maximize, RefreshCcw, ChevronDown, ChevronUp } from 'lucide-react';

// KaTeX渲染组件
const MathDisplay = ({ tex, isError }) => {
  const containerRef = useRef(null);

  useEffect(() => {
    if (window.katex && containerRef.current && tex && !isError) {
      try {
        window.katex.render(tex, containerRef.current, {
          throwOnError: false,
          displayMode: true,
        });
      } catch (e) {
        containerRef.current.innerText = tex;
      }
    } else if (containerRef.current) {
      containerRef.current.innerText = isError ? "无效表达式" : tex;
    }
  }, [tex, isError]);

  return (
    <div
      ref={containerRef}
      className={`text-2xl font-serif flex items-center justify-center min-h-[4rem] overflow-x-auto whitespace-nowrap p-2 ${
        isError ? 'text-red-500 text-lg' : 'text-slate-800'
      }`}
    />
  );
};

export default function App() {
  const [isReady, setIsReady] = useState(false);
  const [expression, setExpression] = useState('');
  const [result, setResult] = useState('');
  const [isError, setIsError] = useState(false);
  const [latex, setLatex] = useState('');
  
  // 绘图相关状态
  const [functions, setFunctions] = useState([]);
  const [offsetX, setOffsetX] = useState(0);
  const [offsetY, setOffsetY] = useState(0);
  const [scale, setScale] = useState(40); // px per unit
  const [mobileTab, setMobileTab] = useState('calc'); // 'calc' or 'graph'
  const [showAdvanced, setShowAdvanced] = useState(true);

  const canvasContainerRef = useRef(null);
  const canvasRef = useRef(null);
  const [dimensions, setDimensions] = useState({ w: 0, h: 0 });
  const [isDragging, setIsDragging] = useState(false);
  const [dragStart, setDragStart] = useState({ x: 0, y: 0 });

  const COLORS = ['#3b82f6', '#ef4444', '#10b981', '#f59e0b', '#8b5cf6'];

  // 动态加载外部依赖 (Math.js 和 KaTeX)
  useEffect(() => {
    const loadScript = (src) => new Promise((resolve, reject) => {
      const s = document.createElement('script');
      s.src = src;
      s.onload = resolve;
      s.onerror = reject;
      document.head.appendChild(s);
    });

    const loadCSS = (href) => {
      const l = document.createElement('link');
      l.rel = 'stylesheet';
      l.href = href;
      document.head.appendChild(l);
    };

    loadCSS("https://cdn.jsdelivr.net/npm/katex@0.16.8/dist/katex.min.css");

    Promise.all([
      loadScript("https://cdnjs.cloudflare.com/ajax/libs/mathjs/11.8.0/math.min.js"),
      loadScript("https://cdn.jsdelivr.net/npm/katex@0.16.8/dist/katex.min.js")
    ]).then(() => setIsReady(true)).catch(console.error);
  }, []);

  // 监听输入并更新LaTeX和结果
  useEffect(() => {
    if (!isReady || !window.math) return;
    
    if (!expression) {
      setLatex('');
      setResult('');
      setIsError(false);
      return;
    }

    try {
      const node = window.math.parse(expression);
      setLatex(node.toTex());
      setIsError(false);

      try {
        const res = window.math.evaluate(expression);
        if (typeof res === 'number') {
          setResult('= ' + window.math.format(res, { precision: 10 }));
        } else {
          setResult('= ' + res.toString());
        }
      } catch (evalErr) {
        if (evalErr.message.includes('Undefined symbol')) {
          setResult('包含变量的函数表达式');
        } else {
          setResult('');
        }
      }
    } catch (parseErr) {
      setLatex(expression); // Fallback
      setIsError(true);
      setResult('');
    }
  }, [expression, isReady]);

  // 响应式画布尺寸
  useEffect(() => {
    if (!canvasContainerRef.current) return;
    const observer = new ResizeObserver(entries => {
      for (let entry of entries) {
        setDimensions({ w: entry.contentRect.width, h: entry.contentRect.height });
      }
    });
    observer.observe(canvasContainerRef.current);
    return () => observer.disconnect();
  }, []);

  // 绘制引擎
  const draw = useCallback(() => {
    if (!canvasRef.current || dimensions.w === 0 || !window.math) return;
    
    const canvas = canvasRef.current;
    const ctx = canvas.getContext('2d');
    const dpr = window.devicePixelRatio || 1;
    const w = dimensions.w;
    const h = dimensions.h;

    canvas.width = w * dpr;
    canvas.height = h * dpr;
    canvas.style.width = `${w}px`;
    canvas.style.height = `${h}px`;
    ctx.scale(dpr, dpr);

    ctx.clearRect(0, 0, w, h);

    // 坐标转换函数
    const toCanvasX = (mx) => (mx - offsetX) * scale + w / 2;
    const toCanvasY = (my) => -(my - offsetY) * scale + h / 2;
    const toMathX = (cx) => (cx - w / 2) / scale + offsetX;
    const toMathY = (cy) => -(cy - h / 2) / scale + offsetY;

    // 绘制网格
    ctx.strokeStyle = '#e2e8f0';
    ctx.lineWidth = 1;
    let step = 1;
    if (scale < 15) step = 5;
    if (scale < 5) step = 10;
    if (scale > 80) step = 0.5;

    const startX = Math.floor(toMathX(0) / step) * step;
    const endX = Math.ceil(toMathX(w) / step) * step;
    for (let x = startX; x <= endX; x += step) {
      const cx = toCanvasX(x);
      ctx.beginPath(); ctx.moveTo(cx, 0); ctx.lineTo(cx, h); ctx.stroke();
      if (x !== 0) {
        ctx.fillStyle = '#94a3b8';
        ctx.font = '10px sans-serif';
        ctx.fillText(Number(x.toPrecision(4)), cx + 4, toCanvasY(0) + 12);
      }
    }

    const startY = Math.floor(toMathY(h) / step) * step;
    const endY = Math.ceil(toMathY(0) / step) * step;
    for (let y = startY; y <= endY; y += step) {
      const cy = toCanvasY(y);
      ctx.beginPath(); ctx.moveTo(0, cy); ctx.lineTo(w, cy); ctx.stroke();
      if (y !== 0) {
        ctx.fillStyle = '#94a3b8';
        ctx.fillText(Number(y.toPrecision(4)), toCanvasX(0) + 4, cy - 4);
      }
    }

    // 绘制坐标轴
    ctx.strokeStyle = '#475569';
    ctx.lineWidth = 2;
    const y0 = toCanvasY(0);
    if (y0 >= 0 && y0 <= h) {
      ctx.beginPath(); ctx.moveTo(0, y0); ctx.lineTo(w, y0); ctx.stroke();
    }
    const x0 = toCanvasX(0);
    if (x0 >= 0 && x0 <= w) {
      ctx.beginPath(); ctx.moveTo(x0, 0); ctx.lineTo(x0, h); ctx.stroke();
    }

    // 绘制所有函数
    functions.forEach(func => {
      try {
        const node = window.math.parse(func.expr);
        const code = node.compile();

        ctx.beginPath();
        ctx.strokeStyle = func.color;
        ctx.lineWidth = 2;
        ctx.lineJoin = 'round';

        let first = true;
        for (let cx = 0; cx <= w; cx += 2) {
          const mx = toMathX(cx);
          try {
            const my = code.evaluate({ x: mx, e: Math.E, pi: Math.PI });
            if (typeof my === 'number' && !isNaN(my)) {
              const cy = toCanvasY(my);
              // 防止跨越渐近线绘制（例如tan(x)）
              if (cy < -2000 || cy > h + 2000) {
                first = true;
                continue;
              }
              if (first) { ctx.moveTo(cx, cy); first = false; }
              else { ctx.lineTo(cx, cy); }
            } else {
              first = true;
            }
          } catch (e) {
            first = true;
          }
        }
        ctx.stroke();
      } catch (e) { }
    });
  }, [dimensions, offsetX, offsetY, scale, functions]);

  useEffect(() => {
    draw();
  }, [draw]);

  // 画布交互事件
  const handleWheel = (e) => {
    e.preventDefault();
    const zoomFactor = e.deltaY > 0 ? 0.9 : 1.1;
    setScale(prev => Math.max(1, Math.min(prev * zoomFactor, 500)));
  };

  const handlePointerDown = (e) => {
    setIsDragging(true);
    setDragStart({ x: e.clientX, y: e.clientY });
  };

  const handlePointerMove = (e) => {
    if (!isDragging) return;
    const dx = e.clientX - dragStart.x;
    const dy = e.clientY - dragStart.y;
    setOffsetX(prev => prev - dx / scale);
    setOffsetY(prev => prev + dy / scale);
    setDragStart({ x: e.clientX, y: e.clientY });
  };

  const handlePointerUp = () => setIsDragging(false);

  // 键盘与操作逻辑
  const handleKey = (key) => {
    if (key === 'C') {
      setExpression('');
      setResult('');
    } else if (key === 'DEL') {
      setExpression(prev => prev.slice(0, -1));
    } else if (key === '=') {
      // 触发强制计算
    } else if (key === 'Plot') {
      if (expression && !isError) {
        setFunctions(prev => {
          // 避免重复绘制
          if (prev.find(f => f.expr === expression)) return prev;
          const color = COLORS[prev.length % COLORS.length];
          return [...prev, { id: Date.now(), expr: expression, color }];
        });
        if (window.innerWidth < 768) setMobileTab('graph');
      }
    } else if (['sin', 'cos', 'tan', 'asin', 'acos', 'atan', 'sinh', 'cosh', 'tanh', 'sqrt', 'log', 'ln', 'abs'].includes(key)) {
      if (key === 'ln') {
        setExpression(prev => prev + 'log('); // Math.js 使用 log(x) 作为自然对数
      } else if (key === 'log') {
        setExpression(prev => prev + 'log10('); // Math.js 使用 log10(x) 作为以10为底
      } else {
        setExpression(prev => prev + key + '(');
      }
    } else if (key === '!') {
      setExpression(prev => prev + '!');
    } else if (key === 'x^2') {
      setExpression(prev => prev + '^2');
    } else if (key === 'x^y') {
      setExpression(prev => prev + '^');
    } else {
      setExpression(prev => prev + key);
    }
  };

  const removeFunction = (id) => {
    setFunctions(prev => prev.filter(f => f.id !== id));
  };

  const resetView = () => {
    setOffsetX(0);
    setOffsetY(0);
    setScale(40);
  };

  const btnClass = "flex items-center justify-center w-full h-full min-h-[2.5rem] sm:min-h-[2.75rem] md:min-h-[3rem] rounded-lg sm:rounded-xl text-sm sm:text-base font-medium transition-all active:scale-95 touch-manipulation";
  const numClass = `${btnClass} bg-white text-slate-800 shadow-sm border border-slate-100 hover:bg-slate-50`;
  const opClass = `${btnClass} bg-indigo-50 text-indigo-600 shadow-sm border border-indigo-100 hover:bg-indigo-100`;
  const funcClass = `${btnClass} bg-slate-100 text-slate-700 shadow-sm hover:bg-slate-200 text-[13px] sm:text-sm`;
  const actionClass = `${btnClass} bg-indigo-500 text-white shadow-md hover:bg-indigo-600`;

  if (!isReady) return (
    <div className="min-h-screen flex items-center justify-center bg-slate-50">
      <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-indigo-500"></div>
    </div>
  );

  return (
    <div className="flex flex-col md:flex-row h-screen w-full bg-slate-50 overflow-hidden font-sans">
      
      {/* 移动端标签页切换 */}
      <div className="md:hidden flex bg-white border-b border-slate-200 shrink-0">
        <button 
          onClick={() => setMobileTab('calc')}
          className={`flex-1 flex items-center justify-center gap-2 py-3 font-medium transition-colors ${mobileTab === 'calc' ? 'text-indigo-600 border-b-2 border-indigo-600' : 'text-slate-500'}`}
        >
          <Calculator size={18} /> 计算
        </button>
        <button 
          onClick={() => setMobileTab('graph')}
          className={`flex-1 flex items-center justify-center gap-2 py-3 font-medium transition-colors ${mobileTab === 'graph' ? 'text-indigo-600 border-b-2 border-indigo-600' : 'text-slate-500'}`}
        >
          <Activity size={18} /> 图表 {functions.length > 0 && <span className="bg-indigo-100 text-indigo-600 text-xs px-2 py-0.5 rounded-full">{functions.length}</span>}
        </button>
      </div>

      {/* 左侧计算面板 */}
      <div className={`w-full md:w-[420px] flex-col border-r border-slate-200 bg-white shadow-[2px_0_10px_rgba(0,0,0,0.02)] z-10 shrink-0 ${mobileTab === 'calc' ? 'flex h-[calc(100vh-50px)] md:h-screen' : 'hidden md:flex h-screen'}`}>
        
        {/* 显示区域 */}
        <div className="flex-none p-6 border-b border-slate-100 bg-slate-50/50 flex flex-col justify-end min-h-[160px]">
          <div className="text-slate-400 text-sm mb-2 text-right h-6">
            {expression || '等待输入...'}
          </div>
          <div className="flex-1 w-full flex items-center justify-end overflow-x-auto">
             <MathDisplay tex={latex} isError={isError} />
          </div>
          <div className="text-indigo-600 font-semibold text-xl text-right h-8 mt-2">
            {result}
          </div>
        </div>

        {/* 绘图函数列表（如果在计算器页也有空间则展示） */}
        {functions.length > 0 && (
          <div className="px-4 py-3 flex gap-2 overflow-x-auto border-b border-slate-100 bg-white items-center shrink-0 hide-scrollbar">
            <span className="text-xs font-semibold text-slate-400 uppercase tracking-wider">已绘图</span>
            {functions.map(f => (
              <div key={f.id} className="flex items-center gap-1.5 px-3 py-1.5 rounded-full bg-slate-50 border border-slate-200 shadow-sm shrink-0">
                <div className="w-2.5 h-2.5 rounded-full" style={{ backgroundColor: f.color }}></div>
                <span className="text-sm font-medium">{f.expr}</span>
                <button onClick={() => removeFunction(f.id)} className="text-slate-400 hover:text-red-500 transition-colors p-0.5">
                  <Trash2 size={14} />
                </button>
              </div>
            ))}
          </div>
        )}

        {/* 高级函数折叠开关 */}
        <div className="flex justify-between items-center px-4 py-2 bg-slate-50/80 border-b border-slate-100 shrink-0">
          <span className="text-xs font-semibold text-slate-400 tracking-wider">数学键盘</span>
          <button 
            onClick={() => setShowAdvanced(!showAdvanced)} 
            className="text-xs text-indigo-600 flex items-center gap-1 hover:text-indigo-700 transition-colors bg-indigo-50 px-2 py-1 rounded-md"
          >
            {showAdvanced ? <><ChevronDown size={14}/> 收起高级函数</> : <><ChevronUp size={14}/> 展开高级函数</>}
          </button>
        </div>

        {/* 键盘区域 */}
        <div className="p-2 sm:p-4 grid grid-cols-5 gap-1.5 sm:gap-2.5 mt-auto flex-1 bg-white overflow-y-auto hide-scrollbar">
          
          {showAdvanced && (
            <>
              <button onClick={() => handleKey('asin')} className={funcClass}>asin</button>
              <button onClick={() => handleKey('acos')} className={funcClass}>acos</button>
              <button onClick={() => handleKey('atan')} className={funcClass}>atan</button>
              <button onClick={() => handleKey('log')} className={funcClass}>log</button>
              <button onClick={() => handleKey('ln')} className={funcClass}>ln</button>
              
              <button onClick={() => handleKey('sinh')} className={funcClass}>sinh</button>
              <button onClick={() => handleKey('cosh')} className={funcClass}>cosh</button>
              <button onClick={() => handleKey('tanh')} className={funcClass}>tanh</button>
              <button onClick={() => handleKey('abs')} className={funcClass}>|x|</button>
              <button onClick={() => handleKey('!')} className={funcClass}>n!</button>
            </>
          )}

          <button onClick={() => handleKey('sin')} className={funcClass}>sin</button>
          <button onClick={() => handleKey('cos')} className={funcClass}>cos</button>
          <button onClick={() => handleKey('tan')} className={funcClass}>tan</button>
          <button onClick={() => handleKey('pi')} className={funcClass}>π</button>
          <button onClick={() => handleKey('C')} className={`${funcClass} text-red-500 font-bold bg-red-50 hover:bg-red-100`}>C</button>
          
          <button onClick={() => handleKey('sqrt')} className={funcClass}>√</button>
          <button onClick={() => handleKey('^')} className={funcClass}>x^y</button>
          <button onClick={() => handleKey('x')} className={`${funcClass} font-bold text-indigo-600 bg-indigo-50`}>x</button>
          <button onClick={() => handleKey('e')} className={funcClass}>e</button>
          <button onClick={() => handleKey('DEL')} className={`${funcClass} text-slate-500`}><Delete size={18}/></button>
          
          <button onClick={() => handleKey('(')} className={funcClass}>(</button>
          <button onClick={() => handleKey(')')} className={funcClass}>)</button>
          <button onClick={() => handleKey('/')} className={opClass}>÷</button>
          <button onClick={() => handleKey('*')} className={opClass}>×</button>
          <button onClick={() => handleKey('-')} className={opClass}>−</button>
          
          <button onClick={() => handleKey('7')} className={numClass}>7</button>
          <button onClick={() => handleKey('8')} className={numClass}>8</button>
          <button onClick={() => handleKey('9')} className={numClass}>9</button>
          <button onClick={() => handleKey('+')} className={`${opClass} row-span-2`}>+</button>
          <button onClick={() => handleKey('Plot')} className={`${actionClass} row-span-2 flex flex-col gap-1 shadow-indigo-200`}>
            <Activity size={18} /> <span className="text-[11px] sm:text-sm">绘制</span>
          </button>
          
          <button onClick={() => handleKey('4')} className={numClass}>4</button>
          <button onClick={() => handleKey('5')} className={numClass}>5</button>
          <button onClick={() => handleKey('6')} className={numClass}>6</button>
          
          <button onClick={() => handleKey('1')} className={numClass}>1</button>
          <button onClick={() => handleKey('2')} className={numClass}>2</button>
          <button onClick={() => handleKey('3')} className={numClass}>3</button>
          <button onClick={() => handleKey('=')} className={`${opClass} row-span-2 font-bold text-xl sm:text-2xl`}>=</button>
          <button onClick={() => setFunctions([])} className={`${funcClass} row-span-2 text-slate-500 flex flex-col gap-1`}>
            <RefreshCcw size={16}/> <span className="text-[10px] sm:text-xs">清空</span>
          </button>
          
          <button onClick={() => handleKey('0')} className={`${numClass} col-span-2`}>0</button>
          <button onClick={() => handleKey('.')} className={numClass}>.</button>
        </div>
      </div>

      {/* 右侧绘图面板 */}
      <div className={`flex-1 relative bg-slate-50 ${mobileTab === 'graph' ? 'flex h-[calc(100vh-50px)] md:h-screen' : 'hidden md:flex h-screen'}`}>
        {/* 画布容器 */}
        <div 
          ref={canvasContainerRef} 
          className="w-full h-full cursor-grab active:cursor-grabbing touch-none"
          onWheel={handleWheel}
          onPointerDown={handlePointerDown}
          onPointerMove={handlePointerMove}
          onPointerUp={handlePointerUp}
          onPointerLeave={handlePointerUp}
        >
          <canvas ref={canvasRef} className="block w-full h-full" />
        </div>

        {/* 绘图控件悬浮窗 */}
        <div className="absolute top-4 right-4 flex flex-col gap-2 bg-white/90 backdrop-blur-sm p-2 rounded-xl shadow-sm border border-slate-200">
          <button onClick={() => setScale(s => Math.min(s * 1.2, 500))} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="放大">
            <ZoomIn size={20} />
          </button>
          <button onClick={() => setScale(s => Math.max(s * 0.8, 1))} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="缩小">
            <ZoomOut size={20} />
          </button>
          <div className="w-full h-px bg-slate-200 my-1"></div>
          <button onClick={resetView} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="重置视图">
            <Maximize size={20} />
          </button>
        </div>

        {functions.length === 0 && (
          <div className="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 text-slate-400 flex flex-col items-center gap-3 pointer-events-none text-center px-4">
            <Activity size={48} className="text-slate-300" opacity={0.5} />
            <p className="text-lg font-medium">还没有绘制任何函数</p>
            <p className="text-sm">在左侧输入包含变量 <span className="font-mono bg-slate-200 px-1.5 py-0.5 rounded text-slate-600">x</span> 的表达式并点击"```react
import React, { useState, useEffect, useRef, useCallback } from 'react';
import { Delete, Trash2, ZoomIn, ZoomOut, Calculator, Activity, Maximize, RefreshCcw, ChevronDown, ChevronUp } from 'lucide-react';

// KaTeX渲染组件
const MathDisplay = ({ tex, isError }) => {
  const containerRef = useRef(null);

  useEffect(() => {
    if (window.katex && containerRef.current && tex && !isError) {
      try {
        window.katex.render(tex, containerRef.current, {
          throwOnError: false,
          displayMode: true,
        });
      } catch (e) {
        containerRef.current.innerText = tex;
      }
    } else if (containerRef.current) {
      containerRef.current.innerText = isError ? "无效表达式" : tex;
    }
  }, [tex, isError]);

  return (
    <div
      ref={containerRef}
      className={`text-2xl font-serif flex items-center justify-center min-h-[4rem] overflow-x-auto whitespace-nowrap p-2 ${
        isError ? 'text-red-500 text-lg' : 'text-slate-800'
      }`}
    />
  );
};

export default function App() {
  const [isReady, setIsReady] = useState(false);
  const [expression, setExpression] = useState('');
  const [result, setResult] = useState('');
  const [isError, setIsError] = useState(false);
  const [latex, setLatex] = useState('');
  
  // 绘图相关状态
  const [functions, setFunctions] = useState([]);
  const [offsetX, setOffsetX] = useState(0);
  const [offsetY, setOffsetY] = useState(0);
  const [scale, setScale] = useState(40); // px per unit
  const [mobileTab, setMobileTab] = useState('calc'); // 'calc' or 'graph'
  const [showAdvanced, setShowAdvanced] = useState(true);

  const canvasContainerRef = useRef(null);
  const canvasRef = useRef(null);
  const [dimensions, setDimensions] = useState({ w: 0, h: 0 });
  const [isDragging, setIsDragging] = useState(false);
  const [dragStart, setDragStart] = useState({ x: 0, y: 0 });

  const COLORS = ['#3b82f6', '#ef4444', '#10b981', '#f59e0b', '#8b5cf6'];

  // 动态加载外部依赖 (Math.js 和 KaTeX)
  useEffect(() => {
    const loadScript = (src) => new Promise((resolve, reject) => {
      const s = document.createElement('script');
      s.src = src;
      s.onload = resolve;
      s.onerror = reject;
      document.head.appendChild(s);
    });

    const loadCSS = (href) => {
      const l = document.createElement('link');
      l.rel = 'stylesheet';
      l.href = href;
      document.head.appendChild(l);
    };

    loadCSS("https://cdn.jsdelivr.net/npm/katex@0.16.8/dist/katex.min.css");

    Promise.all([
      loadScript("https://cdnjs.cloudflare.com/ajax/libs/mathjs/11.8.0/math.min.js"),
      loadScript("https://cdn.jsdelivr.net/npm/katex@0.16.8/dist/katex.min.js")
    ]).then(() => setIsReady(true)).catch(console.error);
  }, []);

  // 监听输入并更新LaTeX和结果
  useEffect(() => {
    if (!isReady || !window.math) return;
    
    if (!expression) {
      setLatex('');
      setResult('');
      setIsError(false);
      return;
    }

    try {
      const node = window.math.parse(expression);
      setLatex(node.toTex());
      setIsError(false);

      try {
        const res = window.math.evaluate(expression);
        if (typeof res === 'number') {
          setResult('= ' + window.math.format(res, { precision: 10 }));
        } else {
          setResult('= ' + res.toString());
        }
      } catch (evalErr) {
        if (evalErr.message.includes('Undefined symbol')) {
          setResult('包含变量的函数表达式');
        } else {
          setResult('');
        }
      }
    } catch (parseErr) {
      setLatex(expression); // Fallback
      setIsError(true);
      setResult('');
    }
  }, [expression, isReady]);

  // 响应式画布尺寸
  useEffect(() => {
    if (!canvasContainerRef.current) return;
    const observer = new ResizeObserver(entries => {
      for (let entry of entries) {
        setDimensions({ w: entry.contentRect.width, h: entry.contentRect.height });
      }
    });
    observer.observe(canvasContainerRef.current);
    return () => observer.disconnect();
  }, []);

  // 绘制引擎
  const draw = useCallback(() => {
    if (!canvasRef.current || dimensions.w === 0 || !window.math) return;
    
    const canvas = canvasRef.current;
    const ctx = canvas.getContext('2d');
    const dpr = window.devicePixelRatio || 1;
    const w = dimensions.w;
    const h = dimensions.h;

    canvas.width = w * dpr;
    canvas.height = h * dpr;
    canvas.style.width = `${w}px`;
    canvas.style.height = `${h}px`;
    ctx.scale(dpr, dpr);

    ctx.clearRect(0, 0, w, h);

    // 坐标转换函数
    const toCanvasX = (mx) => (mx - offsetX) * scale + w / 2;
    const toCanvasY = (my) => -(my - offsetY) * scale + h / 2;
    const toMathX = (cx) => (cx - w / 2) / scale + offsetX;
    const toMathY = (cy) => -(cy - h / 2) / scale + offsetY;

    // 绘制网格
    ctx.strokeStyle = '#e2e8f0';
    ctx.lineWidth = 1;
    let step = 1;
    if (scale < 15) step = 5;
    if (scale < 5) step = 10;
    if (scale > 80) step = 0.5;

    const startX = Math.floor(toMathX(0) / step) * step;
    const endX = Math.ceil(toMathX(w) / step) * step;
    for (let x = startX; x <= endX; x += step) {
      const cx = toCanvasX(x);
      ctx.beginPath(); ctx.moveTo(cx, 0); ctx.lineTo(cx, h); ctx.stroke();
      if (x !== 0) {
        ctx.fillStyle = '#94a3b8';
        ctx.font = '10px sans-serif';
        ctx.fillText(Number(x.toPrecision(4)), cx + 4, toCanvasY(0) + 12);
      }
    }

    const startY = Math.floor(toMathY(h) / step) * step;
    const endY = Math.ceil(toMathY(0) / step) * step;
    for (let y = startY; y <= endY; y += step) {
      const cy = toCanvasY(y);
      ctx.beginPath(); ctx.moveTo(0, cy); ctx.lineTo(w, cy); ctx.stroke();
      if (y !== 0) {
        ctx.fillStyle = '#94a3b8';
        ctx.fillText(Number(y.toPrecision(4)), toCanvasX(0) + 4, cy - 4);
      }
    }

    // 绘制坐标轴
    ctx.strokeStyle = '#475569';
    ctx.lineWidth = 2;
    const y0 = toCanvasY(0);
    if (y0 >= 0 && y0 <= h) {
      ctx.beginPath(); ctx.moveTo(0, y0); ctx.lineTo(w, y0); ctx.stroke();
    }
    const x0 = toCanvasX(0);
    if (x0 >= 0 && x0 <= w) {
      ctx.beginPath(); ctx.moveTo(x0, 0); ctx.lineTo(x0, h); ctx.stroke();
    }

    // 绘制所有函数
    functions.forEach(func => {
      try {
        const node = window.math.parse(func.expr);
        const code = node.compile();

        ctx.beginPath();
        ctx.strokeStyle = func.color;
        ctx.lineWidth = 2;
        ctx.lineJoin = 'round';

        let first = true;
        for (let cx = 0; cx <= w; cx += 2) {
          const mx = toMathX(cx);
          try {
            const my = code.evaluate({ x: mx, e: Math.E, pi: Math.PI });
            if (typeof my === 'number' && !isNaN(my)) {
              const cy = toCanvasY(my);
              // 防止跨越渐近线绘制（例如tan(x)）
              if (cy < -2000 || cy > h + 2000) {
                first = true;
                continue;
              }
              if (first) { ctx.moveTo(cx, cy); first = false; }
              else { ctx.lineTo(cx, cy); }
            } else {
              first = true;
            }
          } catch (e) {
            first = true;
          }
        }
        ctx.stroke();
      } catch (e) { }
    });
  }, [dimensions, offsetX, offsetY, scale, functions]);

  useEffect(() => {
    draw();
  }, [draw]);

  // 画布交互事件
  const handleWheel = (e) => {
    e.preventDefault();
    const zoomFactor = e.deltaY > 0 ? 0.9 : 1.1;
    setScale(prev => Math.max(1, Math.min(prev * zoomFactor, 500)));
  };

  const handlePointerDown = (e) => {
    setIsDragging(true);
    setDragStart({ x: e.clientX, y: e.clientY });
  };

  const handlePointerMove = (e) => {
    if (!isDragging) return;
    const dx = e.clientX - dragStart.x;
    const dy = e.clientY - dragStart.y;
    setOffsetX(prev => prev - dx / scale);
    setOffsetY(prev => prev + dy / scale);
    setDragStart({ x: e.clientX, y: e.clientY });
  };

  const handlePointerUp = () => setIsDragging(false);

  // 键盘与操作逻辑
  const handleKey = (key) => {
    if (key === 'C') {
      setExpression('');
      setResult('');
    } else if (key === 'DEL') {
      setExpression(prev => prev.slice(0, -1));
    } else if (key === '=') {
      // 触发强制计算
    } else if (key === 'Plot') {
      if (expression && !isError) {
        setFunctions(prev => {
          // 避免重复绘制
          if (prev.find(f => f.expr === expression)) return prev;
          const color = COLORS[prev.length % COLORS.length];
          return [...prev, { id: Date.now(), expr: expression, color }];
        });
        if (window.innerWidth < 768) setMobileTab('graph');
      }
    } else if (['sin', 'cos', 'tan', 'asin', 'acos', 'atan', 'sinh', 'cosh', 'tanh', 'sqrt', 'log', 'ln', 'abs'].includes(key)) {
      if (key === 'ln') {
        setExpression(prev => prev + 'log('); // Math.js 使用 log(x) 作为自然对数
      } else if (key === 'log') {
        setExpression(prev => prev + 'log10('); // Math.js 使用 log10(x) 作为以10为底
      } else {
        setExpression(prev => prev + key + '(');
      }
    } else if (key === '!') {
      setExpression(prev => prev + '!');
    } else if (key === 'x^2') {
      setExpression(prev => prev + '^2');
    } else if (key === 'x^y') {
      setExpression(prev => prev + '^');
    } else {
      setExpression(prev => prev + key);
    }
  };

  const removeFunction = (id) => {
    setFunctions(prev => prev.filter(f => f.id !== id));
  };

  const resetView = () => {
    setOffsetX(0);
    setOffsetY(0);
    setScale(40);
  };

  const btnClass = "flex items-center justify-center w-full h-full min-h-[2.5rem] sm:min-h-[2.75rem] md:min-h-[3rem] rounded-lg sm:rounded-xl text-sm sm:text-base font-medium transition-all active:scale-95 touch-manipulation";
  const numClass = `${btnClass} bg-white text-slate-800 shadow-sm border border-slate-100 hover:bg-slate-50`;
  const opClass = `${btnClass} bg-indigo-50 text-indigo-600 shadow-sm border border-indigo-100 hover:bg-indigo-100`;
  const funcClass = `${btnClass} bg-slate-100 text-slate-700 shadow-sm hover:bg-slate-200 text-[13px] sm:text-sm`;
  const actionClass = `${btnClass} bg-indigo-500 text-white shadow-md hover:bg-indigo-600`;

  if (!isReady) return (
    <div className="min-h-screen flex items-center justify-center bg-slate-50">
      <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-indigo-500"></div>
    </div>
  );

  return (
    <div className="flex flex-col md:flex-row h-screen w-full bg-slate-50 overflow-hidden font-sans">
      
      {/* 移动端标签页切换 */}
      <div className="md:hidden flex bg-white border-b border-slate-200 shrink-0">
        <button 
          onClick={() => setMobileTab('calc')}
          className={`flex-1 flex items-center justify-center gap-2 py-3 font-medium transition-colors ${mobileTab === 'calc' ? 'text-indigo-600 border-b-2 border-indigo-600' : 'text-slate-500'}`}
        >
          <Calculator size={18} /> 计算
        </button>
        <button 
          onClick={() => setMobileTab('graph')}
          className={`flex-1 flex items-center justify-center gap-2 py-3 font-medium transition-colors ${mobileTab === 'graph' ? 'text-indigo-600 border-b-2 border-indigo-600' : 'text-slate-500'}`}
        >
          <Activity size={18} /> 图表 {functions.length > 0 && <span className="bg-indigo-100 text-indigo-600 text-xs px-2 py-0.5 rounded-full">{functions.length}</span>}
        </button>
      </div>

      {/* 左侧计算面板 */}
      <div className={`w-full md:w-[420px] flex-col border-r border-slate-200 bg-white shadow-[2px_0_10px_rgba(0,0,0,0.02)] z-10 shrink-0 ${mobileTab === 'calc' ? 'flex h-[calc(100vh-50px)] md:h-screen' : 'hidden md:flex h-screen'}`}>
        
        {/* 显示区域 */}
        <div className="flex-none p-6 border-b border-slate-100 bg-slate-50/50 flex flex-col justify-end min-h-[160px]">
          <div className="text-slate-400 text-sm mb-2 text-right h-6">
            {expression || '等待输入...'}
          </div>
          <div className="flex-1 w-full flex items-center justify-end overflow-x-auto">
             <MathDisplay tex={latex} isError={isError} />
          </div>
          <div className="text-indigo-600 font-semibold text-xl text-right h-8 mt-2">
            {result}
          </div>
        </div>

        {/* 绘图函数列表（如果在计算器页也有空间则展示） */}
        {functions.length > 0 && (
          <div className="px-4 py-3 flex gap-2 overflow-x-auto border-b border-slate-100 bg-white items-center shrink-0 hide-scrollbar">
            <span className="text-xs font-semibold text-slate-400 uppercase tracking-wider">已绘图</span>
            {functions.map(f => (
              <div key={f.id} className="flex items-center gap-1.5 px-3 py-1.5 rounded-full bg-slate-50 border border-slate-200 shadow-sm shrink-0">
                <div className="w-2.5 h-2.5 rounded-full" style={{ backgroundColor: f.color }}></div>
                <span className="text-sm font-medium">{f.expr}</span>
                <button onClick={() => removeFunction(f.id)} className="text-slate-400 hover:text-red-500 transition-colors p-0.5">
                  <Trash2 size={14} />
                </button>
              </div>
            ))}
          </div>
        )}

        {/* 高级函数折叠开关 */}
        <div className="flex justify-between items-center px-4 py-2 bg-slate-50/80 border-b border-slate-100 shrink-0">
          <span className="text-xs font-semibold text-slate-400 tracking-wider">数学键盘</span>
          <button 
            onClick={() => setShowAdvanced(!showAdvanced)} 
            className="text-xs text-indigo-600 flex items-center gap-1 hover:text-indigo-700 transition-colors bg-indigo-50 px-2 py-1 rounded-md"
          >
            {showAdvanced ? <><ChevronDown size={14}/> 收起高级函数</> : <><ChevronUp size={14}/> 展开高级函数</>}
          </button>
        </div>

        {/* 键盘区域 */}
        <div className="p-2 sm:p-4 grid grid-cols-5 gap-1.5 sm:gap-2.5 mt-auto flex-1 bg-white overflow-y-auto hide-scrollbar">
          
          {showAdvanced && (
            <>
              <button onClick={() => handleKey('asin')} className={funcClass}>asin</button>
              <button onClick={() => handleKey('acos')} className={funcClass}>acos</button>
              <button onClick={() => handleKey('atan')} className={funcClass}>atan</button>
              <button onClick={() => handleKey('log')} className={funcClass}>log</button>
              <button onClick={() => handleKey('ln')} className={funcClass}>ln</button>
              
              <button onClick={() => handleKey('sinh')} className={funcClass}>sinh</button>
              <button onClick={() => handleKey('cosh')} className={funcClass}>cosh</button>
              <button onClick={() => handleKey('tanh')} className={funcClass}>tanh</button>
              <button onClick={() => handleKey('abs')} className={funcClass}>|x|</button>
              <button onClick={() => handleKey('!')} className={funcClass}>n!</button>
            </>
          )}

          <button onClick={() => handleKey('sin')} className={funcClass}>sin</button>
          <button onClick={() => handleKey('cos')} className={funcClass}>cos</button>
          <button onClick={() => handleKey('tan')} className={funcClass}>tan</button>
          <button onClick={() => handleKey('pi')} className={funcClass}>π</button>
          <button onClick={() => handleKey('C')} className={`${funcClass} text-red-500 font-bold bg-red-50 hover:bg-red-100`}>C</button>
          
          <button onClick={() => handleKey('sqrt')} className={funcClass}>√</button>
          <button onClick={() => handleKey('^')} className={funcClass}>x^y</button>
          <button onClick={() => handleKey('x')} className={`${funcClass} font-bold text-indigo-600 bg-indigo-50`}>x</button>
          <button onClick={() => handleKey('e')} className={funcClass}>e</button>
          <button onClick={() => handleKey('DEL')} className={`${funcClass} text-slate-500`}><Delete size={18}/></button>
          
          <button onClick={() => handleKey('(')} className={funcClass}>(</button>
          <button onClick={() => handleKey(')')} className={funcClass}>)</button>
          <button onClick={() => handleKey('/')} className={opClass}>÷</button>
          <button onClick={() => handleKey('*')} className={opClass}>×</button>
          <button onClick={() => handleKey('-')} className={opClass}>−</button>
          
          <button onClick={() => handleKey('7')} className={numClass}>7</button>
          <button onClick={() => handleKey('8')} className={numClass}>8</button>
          <button onClick={() => handleKey('9')} className={numClass}>9</button>
          <button onClick={() => handleKey('+')} className={`${opClass} row-span-2`}>+</button>
          <button onClick={() => handleKey('Plot')} className={`${actionClass} row-span-2 flex flex-col gap-1 shadow-indigo-200`}>
            <Activity size={18} /> <span className="text-[11px] sm:text-sm">绘制</span>
          </button>
          
          <button onClick={() => handleKey('4')} className={numClass}>4</button>
          <button onClick={() => handleKey('5')} className={numClass}>5</button>
          <button onClick={() => handleKey('6')} className={numClass}>6</button>
          
          <button onClick={() => handleKey('1')} className={numClass}>1</button>
          <button onClick={() => handleKey('2')} className={numClass}>2</button>
          <button onClick={() => handleKey('3')} className={numClass}>3</button>
          <button onClick={() => handleKey('=')} className={`${opClass} row-span-2 font-bold text-xl sm:text-2xl`}>=</button>
          <button onClick={() => setFunctions([])} className={`${funcClass} row-span-2 text-slate-500 flex flex-col gap-1`}>
            <RefreshCcw size={16}/> <span className="text-[10px] sm:text-xs">清空</span>
          </button>
          
          <button onClick={() => handleKey('0')} className={`${numClass} col-span-2`}>0</button>
          <button onClick={() => handleKey('.')} className={numClass}>.</button>
        </div>
      </div>

      {/* 右侧绘图面板 */}
      <div cla```react
import React, { useState, useEffect, useRef, useCallback } from 'react';
import { Delete, Trash2, ZoomIn, ZoomOut, Calculator, Activity, Maximize, RefreshCcw, ChevronDown, ChevronUp } from 'lucide-react';

// KaTeX渲染组件
const MathDisplay = ({ tex, isError }) => {
  const containerRef = useRef(null);

  useEffect(() => {
    if (window.katex && containerRef.current && tex && !isError) {
      try {
        window.katex.render(tex, containerRef.current, {
          throwOnError: false,
          displayMode: true,
        });
      } catch (e) {
        containerRef.current.innerText = tex;
      }
    } else if (containerRef.current) {
      containerRef.current.innerText = isError ? "无效表达式" : tex;
    }
  }, [tex, isError]);

  return (
    <div
      ref={containerRef}
      className={`text-2xl font-serif flex items-center justify-center min-h-[4rem] overflow-x-auto whitespace-nowrap p-2 ${
        isError ? 'text-red-500 text-lg' : 'text-slate-800'
      }`}
    />
  );
};

export default function App() {
  const [isReady, setIsReady] = useState(false);
  const [expression, setExpression] = useState('');
  const [result, setResult] = useState('');
  const [isError, setIsError] = useState(false);
  const [latex, setLatex] = useState('');
  
  // 绘图相关状态
  const [functions, setFunctions] = useState([]);
  const [offsetX, setOffsetX] = useState(0);
  const [offsetY, setOffsetY] = useState(0);
  const [scale, setScale] = useState(40); // px per unit
  const [mobileTab, setMobileTab] = useState('calc'); // 'calc' or 'graph'
  const [showAdvanced, setShowAdvanced] = useState(true);

  const canvasContainerRef = useRef(null);
  const canvasRef = useRef(null);
  const [dimensions, setDimensions] = useState({ w: 0, h: 0 });
  const [isDragging, setIsDragging] = useState(false);
  const [dragStart, setDragStart] = useState({ x: 0, y: 0 });

  const COLORS = ['#3b82f6', '#ef4444', '#10b981', '#f59e0b', '#8b5cf6'];

  // 动态加载外部依赖 (Math.js 和 KaTeX)
  useEffect(() => {
    const loadScript = (src) => new Promise((resolve, reject) => {
      const s = document.createElement('script');
      s.src = src;
      s.onload = resolve;
      s.onerror = reject;
      document.head.appendChild(s);
    });

    const loadCSS = (href) => {
      const l = document.createElement('link');
      l.rel = 'stylesheet';
      l.href = href;
      document.head.appendChild(l);
    };

    loadCSS("https://cdn.jsdelivr.net/npm/katex@0.16.8/dist/katex.min.css");

    Promise.all([
      loadScript("https://cdnjs.cloudflare.com/ajax/libs/mathjs/11.8.0/math.min.js"),
      loadScript("https://cdn.jsdelivr.net/npm/katex@0.16.8/dist/katex.min.js")
    ]).then(() => setIsReady(true)).catch(console.error);
  }, []);

  // 监听输入并更新LaTeX和结果
  useEffect(() => {
    if (!isReady || !window.math) return;
    
    if (!expression) {
      setLatex('');
      setResult('');
      setIsError(false);
      return;
    }

    try {
      const node = window.math.parse(expression);
      setLatex(node.toTex());
      setIsError(false);

      try {
        const res = window.math.evaluate(expression);
        if (typeof res === 'number') {
          setResult('= ' + window.math.format(res, { precision: 10 }));
        } else {
          setResult('= ' + res.toString());
        }
      } catch (evalErr) {
        if (evalErr.message.includes('Undefined symbol')) {
          setResult('包含变量的函数表达式');
        } else {
          setResult('');
        }
      }
    } catch (parseErr) {
      setLatex(expression); // Fallback
      setIsError(true);
      setResult('');
    }
  }, [expression, isReady]);

  // 响应式画布尺寸
  useEffect(() => {
    if (!canvasContainerRef.current) return;
    const observer = new ResizeObserver(entries => {
      for (let entry of entries) {
        setDimensions({ w: entry.contentRect.width, h: entry.contentRect.height });
      }
    });
    observer.observe(canvasContainerRef.current);
    return () => observer.disconnect();
  }, []);

  // 绘制引擎
  const draw = useCallback(() => {
    if (!canvasRef.current || dimensions.w === 0 || !window.math) return;
    
    const canvas = canvasRef.current;
    const ctx = canvas.getContext('2d');
    const dpr = window.devicePixelRatio || 1;
    const w = dimensions.w;
    const h = dimensions.h;

    canvas.width = w * dpr;
    canvas.height = h * dpr;
    canvas.style.width = `${w}px`;
    canvas.style.height = `${h}px`;
    ctx.scale(dpr, dpr);

    ctx.clearRect(0, 0, w, h);

    // 坐标转换函数
    const toCanvasX = (mx) => (mx - offsetX) * scale + w / 2;
    const toCanvasY = (my) => -(my - offsetY) * scale + h / 2;
    const toMathX = (cx) => (cx - w / 2) / scale + offsetX;
    const toMathY = (cy) => -(cy - h / 2) / scale + offsetY;

    // 绘制网格
    ctx.strokeStyle = '#e2e8f0';
    ctx.lineWidth = 1;
    let step = 1;
    if (scale < 15) step = 5;
    if (scale < 5) step = 10;
    if (scale > 80) step = 0.5;

    const startX = Math.floor(toMathX(0) / step) * step;
    const endX = Math.ceil(toMathX(w) / step) * step;
    for (let x = startX; x <= endX; x += step) {
      const cx = toCanvasX(x);
      ctx.beginPath(); ctx.moveTo(cx, 0); ctx.lineTo(cx, h); ctx.stroke();
      if (x !== 0) {
        ctx.fillStyle = '#94a3b8';
        ctx.font = '10px sans-serif';
        ctx.fillText(Number(x.toPrecision(4)), cx + 4, toCanvasY(0) + 12);
      }
    }

    const startY = Math.floor(toMathY(h) / step) * step;
    const endY = Math.ceil(toMathY(0) / step) * step;
    for (let y = startY; y <= endY; y += step) {
      const cy = toCanvasY(y);
      ctx.beginPath(); ctx.moveTo(0, cy); ctx.lineTo(w, cy); ctx.stroke();
      if (y !== 0) {
        ctx.fillStyle = '#94a3b8';
        ctx.fillText(Number(y.toPrecision(4)), toCanvasX(0) + 4, cy - 4);
      }
    }

    // 绘制坐标轴
    ctx.strokeStyle = '#475569';
    ctx.lineWidth = 2;
    const y0 = toCanvasY(0);
    if (y0 >= 0 && y0 <= h) {
      ctx.beginPath(); ctx.moveTo(0, y0); ctx.lineTo(w, y0); ctx.stroke();
    }
    const x0 = toCanvasX(0);
    if (x0 >= 0 && x0 <= w) {
      ctx.beginPath(); ctx.moveTo(x0, 0); ctx.lineTo(x0, h); ctx.stroke();
    }

    // 绘制所有函数
    functions.forEach(func => {
      try {
        const node = window.math.parse(func.expr);
        const code = node.compile();

        ctx.beginPath();
        ctx.strokeStyle = func.color;
        ctx.lineWidth = 2;
        ctx.lineJoin = 'round';

        let first = true;
        for (let cx = 0; cx <= w; cx += 2) {
          const mx = toMathX(cx);
          try {
            const my = code.evaluate({ x: mx, e: Math.E, pi: Math.PI });
            if (typeof my === 'number' && !isNaN(my)) {
              const cy = toCanvasY(my);
              // 防止跨越渐近线绘制（例如tan(x)）
              if (cy < -2000 || cy > h + 2000) {
                first = true;
                continue;
              }
              if (first) { ctx.moveTo(cx, cy); first = false; }
              else { ctx.lineTo(cx, cy); }
            } else {
              first = true;
            }
          } catch (e) {
            first = true;
          }
        }
        ctx.stroke();
      } catch (e) { }
    });
  }, [dimensions, offsetX, offsetY, scale, functions]);

  useEffect(() => {
    draw();
  }, [draw]);

  // 画布交互事件
  const handleWheel = (e) => {
    e.preventDefault();
    const zoomFactor = e.deltaY > 0 ? 0.9 : 1.1;
    setScale(prev => Math.max(1, Math.min(prev * zoomFactor, 500)));
  };

  const handlePointerDown = (e) => {
    setIsDragging(true);
    setDragStart({ x: e.clientX, y: e.clientY });
  };

  const handlePointerMove = (e) => {
    if (!isDragging) return;
    const dx = e.clientX - dragStart.x;
    const dy = e.clientY - dragStart.y;
    setOffsetX(prev => prev - dx / scale);
    setOffsetY(prev => prev + dy / scale);
    setDragStart({ x: e.clientX, y: e.clientY });
  };

  const handlePointerUp = () => setIsDragging(false);

  // 键盘与操作逻辑
  const handleKey = (key) => {
    if (key === 'C') {
      setExpression('');
      setResult('');
    } else if (key === 'DEL') {
      setExpression(prev => prev.slice(0, -1));
    } else if (key === '=') {
      // 触发强制计算
    } else if (key === 'Plot') {
      if (expression && !isError) {
        setFunctions(prev => {
          // 避免重复绘制
          if (prev.find(f => f.expr === expression)) return prev;
          const color = COLORS[prev.length % COLORS.length];
          return [...prev, { id: Date.now(), expr: expression, color }];
        });
        if (window.innerWidth < 768) setMobileTab('graph');
      }
    } else if (['sin', 'cos', 'tan', 'asin', 'acos', 'atan', 'sinh', 'cosh', 'tanh', 'sqrt', 'log', 'ln', 'abs'].includes(key)) {
      if (key === 'ln') {
        setExpression(prev => prev + 'log('); // Math.js 使用 log(x) 作为自然对数
      } else if (key === 'log') {
        setExpression(prev => prev + 'log10('); // Math.js 使用 log10(x) 作为以10为底
      } else {
        setExpression(prev => prev + key + '(');
      }
    } else if (key === '!') {
      setExpression(prev => prev + '!');
    } else if (key === 'x^2') {
      setExpression(prev => prev + '^2');
    } else if (key === 'x^y') {
      setExpression(prev => prev + '^');
    } else {
      setExpression(prev => prev + key);
    }
  };

  const removeFunction = (id) => {
    setFunctions(prev => prev.filter(f => f.id !== id));
  };

  const resetView = () => {
    setOffsetX(0);
    setOffsetY(0);
    setScale(40);
  };

  const btnClass = "flex items-center justify-center w-full h-full min-h-[2.5rem] sm:min-h-[2.75rem] md:min-h-[3rem] rounded-lg sm:rounded-xl text-sm sm:text-base font-medium transition-all active:scale-95 touch-manipulation";
  const numClass = `${btnClass} bg-white text-slate-800 shadow-sm border border-slate-100 hover:bg-slate-50`;
  const opClass = `${btnClass} bg-indigo-50 text-indigo-600 shadow-sm border border-indigo-100 hover:bg-indigo-100`;
  const funcClass = `${btnClass} bg-slate-100 text-slate-700 shadow-sm hover:bg-slate-200 text-[13px] sm:text-sm`;
  const actionClass = `${btnClass} bg-indigo-500 text-white shadow-md hover:bg-indigo-600`;

  if (!isReady) return (
    <div className="min-h-screen flex items-center justify-center bg-slate-50">
      <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-indigo-500"></div>
    </div>
  );

  return (
    <div className="flex flex-col md:flex-row h-screen w-full bg-slate-50 overflow-hidden font-sans">
      
      {/* 移动端标签页切换 */}
      <div className="md:hidden flex bg-white border-b border-slate-200 shrink-0">
        <button 
          onClick={() => setMobileTab('calc')}
          className={`flex-1 flex items-center justify-center gap-2 py-3 font-medium transition-colors ${mobileTab === 'calc' ? 'text-indigo-600 border-b-2 border-indigo-600' : 'text-slate-500'}`}
        >
          <Calculator size={18} /> 计算
        </button>
        <button 
          onClick={() => setMobileTab('graph')}
          className={`flex-1 flex items-center justify-center gap-2 py-3 font-medium transition-colors ${mobileTab === 'graph' ? 'text-indigo-600 border-b-2 border-indigo-600' : 'text-slate-500'}`}
        >
          <Activity size={18} /> 图表 {functions.length > 0 && <span className="bg-indigo-100 text-indigo-600 text-xs px-2 py-0.5 rounded-full">{functions.length}</span>}
        </button>
      </div>

      {/* 左侧计算面板 */}
      <div className={`w-full md:w-[420px] flex-col border-r border-slate-200 bg-white shadow-[2px_0_10px_rgba(0,0,0,0.02)] z-10 shrink-0 ${mobileTab === 'calc' ? 'flex h-[calc(100vh-50px)] md:h-screen' : 'hidden md:flex h-screen'}`}>
        
        {/* 显示区域 */}
        <div className="flex-none p-6 border-b border-slate-100 bg-slate-50/50 flex flex-col justify-end min-h-[160px]">
          <div className="text-slate-400 text-sm mb-2 text-right h-6">
            {expression || '等待输入...'}
          </div>
          <div className="flex-1 w-full flex items-center justify-end overflow-x-auto">
             <MathDisplay tex={latex} isError={isError} />
          </div>
          <div className="text-indigo-600 font-semibold text-xl text-right h-8 mt-2">
            {result}
          </div>
        </div>

        {/* 绘图函数列表（如果在计算器页也有空间则展示） */}
        {functions.length > 0 && (
          <div className="px-4 py-3 flex gap-2 overflow-x-auto border-b border-slate-100 bg-white items-center shrink-0 hide-scrollbar">
            <span className="text-xs font-semibold text-slate-400 uppercase tracking-wider">已绘图</span>
            {functions.map(f => (
              <div key={f.id} className="flex items-center gap-1.5 px-3 py-1.5 rounded-full bg-slate-50 border border-slate-200 shadow-sm shrink-0">
                <div className="w-2.5 h-2.5 rounded-full" style={{ backgroundColor: f.color }}></div>
                <span className="text-sm font-medium">{f.expr}</span>
                <button onClick={() => removeFunction(f.id)} className="text-slate-400 hover:text-red-500 transition-colors p-0.5">
                  <Trash2 size={14} />
                </button>
              </div>
            ))}
          </div>
        )}

        {/* 高级函数折叠开关 */}
        <div className="flex justify-between items-center px-4 py-2 bg-slate-50/80 border-b border-slate-100 shrink-0">
          <span className="text-xs font-semibold text-slate-400 tracking-wider">数学键盘</span>
          <button 
            onClick={() => setShowAdvanced(!showAdvanced)} 
            className="text-xs text-indigo-600 flex items-center gap-1 hover:text-indigo-700 transition-colors bg-indigo-50 px-2 py-1 rounded-md"
          >
            {showAdvanced ? <><ChevronDown size={14}/> 收起高级函数</> : <><ChevronUp size={14}/> 展开高级函数</>}
          </button>
        </div>

        {/* 键盘区域 */}
        <div className="p-2 sm:p-4 grid grid-cols-5 gap-1.5 sm:gap-2.5 mt-auto flex-1 bg-white overflow-y-auto hide-scrollbar">
          
          {showAdvanced && (
            <>
              <button onClick={() => handleKey('asin')} className={funcClass}>asin</button>
              <button onClick={() => handleKey('acos')} className={funcClass}>acos</button>
              <button onClick={() => handleKey('atan')} className={funcClass}>atan</button>
              <button onClick={() => handleKey('log')} className={funcClass}>log</button>
              <button onClick={() => handleKey('ln')} className={funcClass}>ln</button>
              
              <button onClick={() => handleKey('sinh')} className={funcClass}>sinh</button>
              <button onClick={() => handleKey('cosh')} className={funcClass}>cosh</button>
              <button onClick={() => handleKey('tanh')} className={funcClass}>tanh</button>
              <button onClick={() => handleKey('abs')} className={funcClass}>|x|</button>
              <button onClick={() => handleKey('!')} className={funcClass}>n!</button>
            </>
          )}

          <button onClick={() => handleKey('sin')} className={funcClass}>sin</button>
          <button onClick={() => handleKey('cos')} className={funcClass}>cos</button>
          <button onClick={() => handleKey('tan')} className={funcClass}>tan</button>
          <button onClick={() => handleKey('pi')} className={funcClass}>π</button>
          <button onClick={() => handleKey('C')} className={`${funcClass} text-red-500 font-bold bg-red-50 hover:bg-red-100`}>C</button>
          
          <button onClick={() => handleKey('sqrt')} className={funcClass}>√</button>
          <button onClick={() => handleKey('^')} className={funcClass}>x^y</button>
          <button onClick={() => handleKey('x')} className={`${funcClass} font-bold text-indigo-600 bg-indigo-50`}>x</button>
          <button onClick={() => handleKey('e')} className={funcClass}>e</button>
          <button onClick={() => handleKey('DEL')} className={`${funcClass} text-slate-500`}><Delete size={18}/></button>
          
          <button onClick={() => handleKey('(')} className={funcClass}>(</button>
          <button onClick={() => handleKey(')')} className={funcClass}>)</button>
          <button onClick={() => handleKey('/')} className={opClass}>÷</button>
          <button onClick={() => handleKey('*')} className={opClass}>×</button>
          <button onClick={() => handleKey('-')} className={opClass}>−</button>
          
          <button onClick={() => handleKey('7')} className={numClass}>7</button>
          <button onClick={() => handleKey('8')} className={numClass}>8</button>
          <button onClick={() => handleKey('9')} className={numClass}>9</button>
          <button onClick={() => handleKey('+')} className={`${opClass} row-span-2`}>+</button>
          <button onClick={() => handleKey('Plot')} className={`${actionClass} row-span-2 flex flex-col gap-1 shadow-indigo-200`}>
            <Activity size={18} /> <span className="text-[11px] sm:text-sm">绘制</span>
          </button>
          
          <button onClick={() => handleKey('4')} className={numClass}>4</button>
          <button onClick={() => handleKey('5')} className={numClass}>5</button>
          <button onClick={() => handleKey('6')} className={numClass}>6</button>
          
          <button onClick={() => handleKey('1')} className={numClass}>1</button>
          <button onClick={() => handleKey('2')} className={numClass}>2</button>
          <button onClick={() => handleKey('3')} className={numClass}>3</button>
          <button onClick={() => handleKey('=')} className={`${opClass} row-span-2 font-bold text-xl sm:text-2xl`}>=</button>
          <button onClick={() => setFunctions([])} className={`${funcClass} row-span-2 text-slate-500 flex flex-col gap-1`}>
            <RefreshCcw size={16}/> <span className="text-[10px] sm:text-xs">清空</span>
          </button>
          
          <button onClick={() => handleKey('0')} className={`${numClass} col-span-2`}>0</button>
          <button onClick={() => handleKey('.')} className={numClass}>.</button>
        </div>
      </div>

      {/* 右侧绘图面板 */}
      <div className={`flex-1 relative bg-slate-50 ${mobileTab === 'graph' ? 'flex h-[calc(100vh-50px)] md:h-screen' : 'hidden md:flex h-screen'}`}>
        {/* 画布容器 */}
        <div 
          ref={canvasContainerRef} 
          className="w-full h-full cursor-grab active:cursor-grabbing touch-none"
          onWheel={handleWheel}
          onPointerDown={handlePointerDown}
          onPointerMove={handlePointerMove}
          onPointerUp={handlePointerUp}
          onPointerLeave={handlePointerUp}
        >
          <canvas ref={canvasRef} className="block w-full h-full" />
        </div>

        {/* 绘图控件悬浮窗 */}
        <div className="absolute top-4 right-4 flex flex-col gap-2 bg-white/90 backdrop-blur-sm p-2 rounded-xl shadow-sm border border-slate-200">
          <button onClick={() => setScale(s => Math.min(s * 1.2, 500))} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="放大">
            <ZoomIn size={20} />
          </button>
          <button onClick={() => setScale(s => Math.max(s * 0.8, 1))} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="缩小">
            <ZoomOut size={20} />
          </button>
          <div className="w-full h-px bg-slate-200 my-1"></div>
          <button onClick={resetView} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="重置视图">
            <Maximize size={20} />
          </button>
        </div>

        {functions.length === 0 && (
          <div className="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 text-slate-400 flex flex-col items-center gap-3 pointer-events-none text-center px-4">
            <Activity size={48} className="text-slate-300" opacity={0.5} />
            <p className="text-lg font-medium">还没有绘制任何函数</p>
            <p className="text-sm">在左侧输入包含变量 <span className="font-mono bg-slate-200 px-1.5 py-0.5 rounded text-slate-600">x</span> 的表达式并点击"```react
import React, { useState, useEffect, useRef, useCallback } from 'react';
import { Delete, Trash2, ZoomIn, ZoomOut, Calculator, Activity, Maximize, RefreshCcw, ChevronDown, ChevronUp } from 'lucide-react';

// KaTeX渲染组件
const MathDisplay = ({ tex, isError }) => {
  const containerRef = useRef(null);

  useEffect(() => {
    if (window.katex && containerRef.current && tex && !isError) {
      try {
        window.katex.render(tex, containerRef.current, {
          throwOnError: false,
          displayMode: true,
        });
      } catch (e) {
        containerRef.current.innerText = tex;
      }
    } else if (containerRef.current) {
      containerRef.current.innerText = isError ? "无效表达式" : tex;
    }
  }, [tex, isError]);

  return (
    <div
      ref={containerRef}
      className={`text-2xl font-serif flex items-center justify-center min-h-[4rem] overflow-x-auto whitespace-nowrap p-2 ${
        isError ? 'text-red-500 text-lg' : 'text-slate-800'
      }`}
    />
  );
};

export default function App() {
  const [isReady, setIsReady] = useState(false);
  const [expression, setExpression] = useState('');
  const [result, setResult] = useState('');
  const [isError, setIsError] = useState(false);
  const [latex, setLatex] = useState('');
  
  // 绘图相关状态
  const [functions, setFunctions] = useState([]);
  const [offsetX, setOffsetX] = useState(0);
  const [offsetY, setOffsetY] = useState(0);
  const [scale, setScale] = useState(40); // px per unit
  const [mobileTab, setMobileTab] = useState('calc'); // 'calc' or 'graph'
  const [showAdvanced, setShowAdvanced] = useState(true);

  const canvasContainerRef = useRef(null);
  const canvasRef = useRef(null);
  const [dimensions, setDimensions] = useState({ w: 0, h: 0 });
  const [isDragging, setIsDragging] = useState(false);
  const [dragStart, setDragStart] = useState({ x: 0, y: 0 });

  const COLORS = ['#3b82f6', '#ef4444', '#10b981', '#f59e0b', '#8b5cf6'];

  // 动态加载外部依赖 (Math.js 和 KaTeX)
  useEffect(() => {
    const loadScript = (src) => new Promise((resolve, reject) => {
      const s = document.createElement('script');
      s.src = src;
      s.onload = resolve;
      s.onerror = reject;
      document.head.appendChild(s);
    });

    const loadCSS = (href) => {
      const l = document.createElement('link');
      l.rel = 'stylesheet';
      l.href = href;
      document.head.appendChild(l);
    };

    loadCSS("https://cdn.jsdelivr.net/npm/katex@0.16.8/dist/katex.min.css");

    Promise.all([
      loadScript("https://cdnjs.cloudflare.com/ajax/libs/mathjs/11.8.0/math.min.js"),
      loadScript("https://cdn.jsdelivr.net/npm/katex@0.16.8/dist/katex.min.js")
    ]).then(() => setIsReady(true)).catch(console.error);
  }, []);

  // 监听输入并更新LaTeX和结果
  useEffect(() => {
    if (!isReady || !window.math) return;
    
    if (!expression) {
      setLatex('');
      setResult('');
      setIsError(false);
      return;
    }

    try {
      const node = window.math.parse(expression);
      setLatex(node.toTex());
      setIsError(false);

      try {
        const res = window.math.evaluate(expression);
        if (typeof res === 'number') {
          setResult('= ' + window.math.format(res, { precision: 10 }));
        } else {
          setResult('= ' + res.toString());
        }
      } catch (evalErr) {
        if (evalErr.message.includes('Undefined symbol')) {
          setResult('包含变量的函数表达式');
        } else {
          setResult('');
        }
      }
    } catch (parseErr) {
      setLatex(expression); // Fallback
      setIsError(true);
      setResult('');
    }
  }, [expression, isReady]);

  // 响应式画布尺寸
  useEffect(() => {
    if (!canvasContainerRef.current) return;
    const observer = new ResizeObserver(entries => {
      for (let entry of entries) {
        setDimensions({ w: entry.contentRect.width, h: entry.contentRect.height });
      }
    });
    observer.observe(canvasContainerRef.current);
    return () => observer.disconnect();
  }, []);

  // 绘制引擎
  const draw = useCallback(() => {
    if (!canvasRef.current || dimensions.w === 0 || !window.math) return;
    
    const canvas = canvasRef.current;
    const ctx = canvas.getContext('2d');
    const dpr = window.devicePixelRatio || 1;
    const w = dimensions.w;
    const h = dimensions.h;

    canvas.width = w * dpr;
    canvas.height = h * dpr;
    canvas.style.width = `${w}px`;
    canvas.style.height = `${h}px`;
    ctx.scale(dpr, dpr);

    ctx.clearRect(0, 0, w, h);

    // 坐标转换函数
    const toCanvasX = (mx) => (mx - offsetX) * scale + w / 2;
    const toCanvasY = (my) => -(my - offsetY) * scale + h / 2;
    const toMathX = (cx) => (cx - w / 2) / scale + offsetX;
    const toMathY = (cy) => -(cy - h / 2) / scale + offsetY;

    // 绘制网格
    ctx.strokeStyle = '#e2e8f0';
    ctx.lineWidth = 1;
    let step = 1;
    if (scale < 15) step = 5;
    if (scale < 5) step = 10;
    if (scale > 80) step = 0.5;

    const startX = Math.floor(toMathX(0) / step) * step;
    const endX = Math.ceil(toMathX(w) / step) * step;
    for (let x = startX; x <= endX; x += step) {
      const cx = toCanvasX(x);
      ctx.beginPath(); ctx.moveTo(cx, 0); ctx.lineTo(cx, h); ctx.stroke();
      if (x !== 0) {
        ctx.fillStyle = '#94a3b8';
        ctx.font = '10px sans-serif';
        ctx.fillText(Number(x.toPrecision(4)), cx + 4, toCanvasY(0) + 12);
      }
    }

    const startY = Math.floor(toMathY(h) / step) * step;
    const endY = Math.ceil(toMathY(0) / step) * step;
    for (let y = startY; y <= endY; y += step) {
      const cy = toCanvasY(y);
      ctx.beginPath(); ctx.moveTo(0, cy); ctx.lineTo(w, cy); ctx.stroke();
      if (y !== 0) {
        ctx.fillStyle = '#94a3b8';
        ctx.fillText(Number(y.toPrecision(4)), toCanvasX(0) + 4, cy - 4);
      }
    }

    // 绘制坐标轴
    ctx.strokeStyle = '#475569';
    ctx.lineWidth = 2;
    const y0 = toCanvasY(0);
    if (y0 >= 0 && y0 <= h) {
      ctx.beginPath(); ctx.moveTo(0, y0); ctx.lineTo(w, y0); ctx.stroke();
    }
    const x0 = toCanvasX(0);
    if (x0 >= 0 && x0 <= w) {
      ctx.beginPath(); ctx.moveTo(x0, 0); ctx.lineTo(x0, h); ctx.stroke();
    }

    // 绘制所有函数
    functions.forEach(func => {
      try {
        const node = window.math.parse(func.expr);
        const code = node.compile();

        ctx.beginPath();
        ctx.strokeStyle = func.color;
        ctx.lineWidth = 2;
        ctx.lineJoin = 'round';

        let first = true;
        for (let cx = 0; cx <= w; cx += 2) {
          const mx = toMathX(cx);
          try {
            const my = code.evaluate({ x: mx, e: Math.E, pi: Math.PI });
            if (typeof my === 'number' && !isNaN(my)) {
              const cy = toCanvasY(my);
              // 防止跨越渐近线绘制（例如tan(x)）
              if (cy < -2000 || cy > h + 2000) {
                first = true;
                continue;
              }
              if (first) { ctx.moveTo(cx, cy); first = false; }
              else { ctx.lineTo(cx, cy); }
            } else {
              first = true;
            }
          } catch (e) {
            first = true;
          }
        }
        ctx.stroke();
      } catch (e) { }
    });
  }, [dimensions, offsetX, offsetY, scale, functions]);

  useEffect(() => {
    draw();
  }, [draw]);

  // 画布交互事件
  const handleWheel = (e) => {
    e.preventDefault();
    const zoomFactor = e.deltaY > 0 ? 0.9 : 1.1;
    setScale(prev => Math.max(1, Math.min(prev * zoomFactor, 500)));
  };

  const handlePointerDown = (e) => {
    setIsDragging(true);
    setDragStart({ x: e.clientX, y: e.clientY });
  };

  const handlePointerMove = (e) => {
    if (!isDragging) return;
    const dx = e.clientX - dragStart.x;
    const dy = e.clientY - dragStart.y;
    setOffsetX(prev => prev - dx / scale);
    setOffsetY(prev => prev + dy / scale);
    setDragStart({ x: e.clientX, y: e.clientY });
  };

  const handlePointerUp = () => setIsDragging(false);

  // 键盘与操作逻辑
  const handleKey = (key) => {
    if (key === 'C') {
      setExpression('');
      setResult('');
    } else if (key === 'DEL') {
      setExpression(prev => prev.slice(0, -1));
    } else if (key === '=') {
      // 触发强制计算
    } else if (key === 'Plot') {
      if (expression && !isError) {
        setFunctions(prev => {
          // 避免重复绘制
          if (prev.find(f => f.expr === expression)) return prev;
          const color = COLORS[prev.length % COLORS.length];
          return [...prev, { id: Date.now(), expr: expression, color }];
        });
        if (window.innerWidth < 768) setMobileTab('graph');
      }
    } else if (['sin', 'cos', 'tan', 'asin', 'acos', 'atan', 'sinh', 'cosh', 'tanh', 'sqrt', 'log', 'ln', 'abs'].includes(key)) {
      if (key === 'ln') {
        setExpression(prev => prev + 'log('); // Math.js 使用 log(x) 作为自然对数
      } else if (key === 'log') {
        setExpression(prev => prev + 'log10('); // Math.js 使用 log10(x) 作为以10为底
      } else {
        setExpression(prev => prev + key + '(');
      }
    } else if (key === '!') {
      setExpression(prev => prev + '!');
    } else if (key === 'x^2') {
      setExpression(prev => prev + '^2');
    } else if (key === 'x^y') {
      setExpression(prev => prev + '^');
    } else {
      setExpression(prev => prev + key);
    }
  };

  const removeFunction = (id) => {
    setFunctions(prev => prev.filter(f => f.id !== id));
  };

  const resetView = () => {
    setOffsetX(0);
    setOffsetY(0);
    setScale(40);
  };

  const btnClass = "flex items-center justify-center w-full h-full min-h-[2.5rem] sm:min-h-[2.75rem] md:min-h-[3rem] rounded-lg sm:rounded-xl text-sm sm:text-base font-medium transition-all active:scale-95 touch-manipulation";
  const numClass = `${btnClass} bg-white text-slate-800 shadow-sm border border-slate-100 hover:bg-slate-50`;
  const opClass = `${btnClass} bg-indigo-50 text-indigo-600 shadow-sm border border-indigo-100 hover:bg-indigo-100`;
  const funcClass = `${btnClass} bg-slate-100 text-slate-700 shadow-sm hover:bg-slate-200 text-[13px] sm:text-sm`;
  const actionClass = `${btnClass} bg-indigo-500 text-white shadow-md hover:bg-indigo-600`;

  if (!isReady) return (
    <div className="min-h-screen flex items-center justify-center bg-slate-50">
      <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-indigo-500"></div>
    </div>
  );

  return (
    <div className="flex flex-col md:flex-row h-screen w-full bg-slate-50 overflow-hidden font-sans">
      
      {/* 移动端标签页切换 */}
      <div className="md:hidden flex bg-white border-b border-slate-200 shrink-0">
        <button 
          onClick={() => setMobileTab('calc')}
          className={`flex-1 flex items-center justify-center gap-2 py-3 font-medium transition-colors ${mobileTab === 'calc' ? 'text-indigo-600 border-b-2 border-indigo-600' : 'text-slate-500'}`}
        >
          <Calculator size={18} /> 计算
        </button>
        <button 
          onClick={() => setMobileTab('graph')}
          className={`flex-1 flex items-center justify-center gap-2 py-3 font-medium transition-colors ${mobileTab === 'graph' ? 'text-indigo-600 border-b-2 border-indigo-600' : 'text-slate-500'}`}
        >
          <Activity size={18} /> 图表 {functions.length > 0 && <span className="bg-indigo-100 text-indigo-600 text-xs px-2 py-0.5 rounded-full">{functions.length}</span>}
        </button>
      </div>

      {/* 左侧计算面板 */}
      <div className={`w-full md:w-[420px] flex-col border-r border-slate-200 bg-white shadow-[2px_0_10px_rgba(0,0,0,0.02)] z-10 shrink-0 ${mobileTab === 'calc' ? 'flex h-[calc(100vh-50px)] md:h-screen' : 'hidden md:flex h-screen'}`}>
        
        {/* 显示区域 */}
        <div className="flex-none p-6 border-b border-slate-100 bg-slate-50/50 flex flex-col justify-end min-h-[160px]">
          <div className="text-slate-400 text-sm mb-2 text-right h-6">
            {expression || '等待输入...'}
          </div>
          <div className="flex-1 w-full flex items-center justify-end overflow-x-auto">
             <MathDisplay tex={latex} isError={isError} />
          </div>
          <div className="text-indigo-600 font-semibold text-xl text-right h-8 mt-2">
            {result}
          </div>
        </div>

        {/* 绘图函数列表（如果在计算器页也有空间则展示） */}
        {functions.length > 0 && (
          <div className="px-4 py-3 flex gap-2 overflow-x-auto border-b border-slate-100 bg-white items-center shrink-0 hide-scrollbar">
            <span className="text-xs font-semibold text-slate-400 uppercase tracking-wider">已绘图</span>
            {functions.map(f => (
              <div key={f.id} className="flex items-center gap-1.5 px-3 py-1.5 rounded-full bg-slate-50 border border-slate-200 shadow-sm shrink-0">
                <div className="w-2.5 h-2.5 rounded-full" style={{ backgroundColor: f.color }}></div>
                <span className="text-sm font-medium">{f.expr}</span>
                <button onClick={() => removeFunction(f.id)} className="text-slate-400 hover:text-red-500 transition-colors p-0.5">
                  <Trash2 size={14} />
                </button>
              </div>
            ))}
          </div>
        )}

        {/* 高级函数折叠开关 */}
        <div className="flex justify-between items-center px-4 py-2 bg-slate-50/80 border-b border-slate-100 shrink-0">
          <span className="text-xs font-semibold text-slate-400 tracking-wider">数学键盘</span>
          <button 
            onClick={() => setShowAdvanced(!showAdvanced)} 
            className="text-xs text-indigo-600 flex items-center gap-1 hover:text-indigo-700 transition-colors bg-indigo-50 px-2 py-1 rounded-md"
          >
            {showAdvanced ? <><ChevronDown size={14}/> 收起高级函数</> : <><ChevronUp size={14}/> 展开高级函数</>}
          </button>
        </div>

        {/* 键盘区域 */}
        <div className="p-2 sm:p-4 grid grid-cols-5 gap-1.5 sm:gap-2.5 mt-auto flex-1 bg-white overflow-y-auto hide-scrollbar">
          
          {showAdvanced && (
            <>
              <button onClick={() => handleKey('asin')} className={funcClass}>asin</button>
              <button onClick={() => handleKey('acos')} className={funcClass}>acos</button>
              <button onClick={() => handleKey('atan')} className={funcClass}>atan</button>
              <button onClick={() => handleKey('log')} className={funcClass}>log</button>
              <button onClick={() => handleKey('ln')} className={funcClass}>ln</button>
              
              <button onClick={() => handleKey('sinh')} className={funcClass}>sinh</button>
              <button onClick={() => handleKey('cosh')} className={funcClass}>cosh</button>
              <button onClick={() => handleKey('tanh')} className={funcClass}>tanh</button>
              <button onClick={() => handleKey('abs')} className={funcClass}>|x|</button>
              <button onClick={() => handleKey('!')} className={funcClass}>n!</button>
            </>
          )}

          <button onClick={() => handleKey('sin')} className={funcClass}>sin</button>
          <button onClick={() => handleKey('cos')} className={funcClass}>cos</button>
          <button onClick={() => handleKey('tan')} className={funcClass}>tan</button>
          <button onClick={() => handleKey('pi')} className={funcClass}>π</button>
          <button onClick={() => handleKey('C')} className={`${funcClass} text-red-500 font-bold bg-red-50 hover:bg-red-100`}>C</button>
          
          <button onClick={() => handleKey('sqrt')} className={funcClass}>√</button>
          <button onClick={() => handleKey('^')} className={funcClass}>x^y</button>
          <button onClick={() => handleKey('x')} className={`${funcClass} font-bold text-indigo-600 bg-indigo-50`}>x</button>
          <button onClick={() => handleKey('e')} className={funcClass}>e</button>
          <button onClick={() => handleKey('DEL')} className={`${funcClass} text-slate-500`}><Delete size={18}/></button>
          
          <button onClick={() => handleKey('(')} className={funcClass}>(</button>
          <button onClick={() => handleKey(')')} className={funcClass}>)</button>
          <button onClick={() => handleKey('/')} className={opClass}>÷</button>
          <button onClick={() => handleKey('*')} className={opClass}>×</button>
          <button onClick={() => handleKey('-')} className={opClass}>−</button>
          
          <button onClick={() => handleKey('7')} className={numClass}>7</button>
          <button onClick={() => handleKey('8')} className={numClass}>8</button>
          <button onClick={() => handleKey('9')} className={numClass}>9</button>
          <button onClick={() => handleKey('+')} className={`${opClass} row-span-2`}>+</button>
          <button onClick={() => handleKey('Plot')} className={`${actionClass} row-span-2 flex flex-col gap-1 shadow-indigo-200`}>
            <Activity size={18} /> <span className="text-[11px] sm:text-sm">绘制</span>
          </button>
          
          <button onClick={() => handleKey('4')} className={numClass}>4</button>
          <button onClick={() => handleKey('5')} className={numClass}>5</button>
          <button onClick={() => handleKey('6')} className={numClass}>6</button>
          
          <button onClick={() => handleKey('1')} className={numClass}>1</button>
          <button onClick={() => handleKey('2')} className={numClass}>2</button>
          <button onClick={() => handleKey('3')} className={numClass}>3</button>
          <button onClick={() => handleKey('=')} className={`${opClass} row-span-2 font-bold text-xl sm:text-2xl`}>=</button>
          <button onClick={() => setFunctions([])} className={`${funcClass} row-span-2 text-slate-500 flex flex-col gap-1`}>
            <RefreshCcw size={16}/> <span className="text-[10px] sm:text-xs">清空</span>
          </button>
          
          <button onClick={() => handleKey('0')} className={`${numClass} col-span-2`}>0</button>
          <button onClick={() => handleKey('.')} className={numClass}>.</button>
        </div>
      </div>

      {/* 右侧绘图面板 */}
      <div className={`flex-1 relative bg-slate-50 ${mobileTab === 'graph' ? 'flex h-[calc(100vh-50px)] md:h-screen' : 'hidden md:flex h-screen'}`}>
        {/* 画布容器 */}
        <div 
          ref={canvasContainerRef} 
          className="w-full h-full cursor-grab active:cursor-grabbing touch-none"
          onWheel={handleWheel}
          onPointerDown={handlePointerDown}
          onPointerMove={handlePointerMove}
          onPointerUp={handlePointerUp}
          onPointerLeave={handlePointerUp}
        >
          <canvas ref={canvasRef} className="block w-full h-full" />
        </div>

        {/* 绘图控件悬浮窗 */}
        <div className="absolute top-4 right-4 flex flex-col gap-2 bg-white/90 backdrop-blur-sm p-2 rounded-xl shadow-sm border border-slate-200">
          <button onClick={() => setScale(s => Math.min(s * 1.2, 500))} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="放大">
            <ZoomIn size={20} />
          </button>
          <button onClick={() => setScale(s => Math.max(s * 0.8, 1))} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="缩小">
            <ZoomOut size={20} />
          </button>
          <div className="w-full h-px bg-slate-200 my-1"></div>
          <button onClick={resetView} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="重置视图">
            <Maximize size={20} />
          </button>
        </div>

        {functions.length === 0 && (
          <div className="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 text-slate-400 flex flex-col items-center gap-3 pointer-events-none text-center px-4">
            <Activity size={48} className="text-slate-300" opacity={0.5} />
            <p className="text-lg font-medium">还没有绘制任何函数</p>
            <p className="text-sm">在左侧输入包含变量 <span className="font-mono bg-slate-200 px-1.5 py-0.5 rounded text-slate-600">x</span> 的表达式并点击"```react
import React, { useState, useEffect, useRef, useCallback } from 'react';
import { Delete, Trash2, ZoomIn, ZoomOut, Calculator, Activity, Maximize, RefreshCcw, ChevronDown, ChevronUp } from 'lucide-react';

// KaTeX渲染组件
const MathDisplay = ({ tex, isError }) => {
  const containerRef = useRef(null);

  useEffect(() => {
    if (window.katex && containerRef.current && tex && !isError) {
      try {
        window.katex.render(tex, containerRef.current, {
          throwOnError: false,
          displayMode: true,
        });
      } catch (e) {
        containerRef.current.innerText = tex;
      }
    } else if (containerRef.current) {
      containerRef.current.innerText = isError ? "无效表达式" : tex;
    }
  }, [tex, isError]);

  return (
    <div
      ref={containerRef}
      className={`text-2xl font-serif flex items-center justify-center min-h-[4rem] overflow-x-auto whitespace-nowrap p-2 ${
        isError ? 'text-red-500 text-lg' : 'text-slate-800'
      }`}
    />
  );
};

export default function App() {
  const [isReady, setIsReady] = useState(false);
  const [expression, setExpression] = useState('');
  const [result, setResult] = useState('');
  const [isError, setIsError] = useState(false);
  const [latex, setLatex] = useState('');
  
  // 绘图相关状态
  const [functions, setFunctions] = useState([]);
  const [offsetX, setOffsetX] = useState(0);
  const [offsetY, setOffsetY] = useState(0);
  const [scale, setScale] = useState(40); // px per unit
  const [mobileTab, setMobileTab] = useState('calc'); // 'calc' or 'graph'
  const [showAdvanced, setShowAdvanced] = useState(true);

  const canvasContainerRef = useRef(null);
  const canvasRef = useRef(null);
  const [dimensions, setDimensions] = useState({ w: 0, h: 0 });
  const [isDragging, setIsDragging] = useState(false);
  const [dragStart, setDragStart] = useState({ x: 0, y: 0 });

  const COLORS = ['#3b82f6', '#ef4444', '#10b981', '#f59e0b', '#8b5cf6'];

  // 动态加载外部依赖 (Math.js 和 KaTeX)
  useEffect(() => {
    const loadScript = (src) => new Promise((resolve, reject) => {
      const s = document.createElement('script');
      s.src = src;
      s.onload = resolve;
      s.onerror = reject;
      document.head.appendChild(s);
    });

    const loadCSS = (href) => {
      const l = document.createElement('link');
      l.rel = 'stylesheet';
      l.href = href;
      document.head.appendChild(l);
    };

    loadCSS("https://cdn.jsdelivr.net/npm/katex@0.16.8/dist/katex.min.css");

    Promise.all([
      loadScript("https://cdnjs.cloudflare.com/ajax/libs/mathjs/11.8.0/math.min.js"),
      loadScript("https://cdn.jsdelivr.net/npm/katex@0.16.8/dist/katex.min.js")
    ]).then(() => setIsReady(true)).catch(console.error);
  }, []);

  // 监听输入并更新LaTeX和结果
  useEffect(() => {
    if (!isReady || !window.math) return;
    
    if (!expression) {
      setLatex('');
      setResult('');
      setIsError(false);
      return;
    }

    try {
      const node = window.math.parse(expression);
      setLatex(node.toTex());
      setIsError(false);

      try {
        const res = window.math.evaluate(expression);
        if (typeof res === 'number') {
          setResult('= ' + window.math.format(res, { precision: 10 }));
        } else {
          setResult('= ' + res.toString());
        }
      } catch (evalErr) {
        if (evalErr.message.includes('Undefined symbol')) {
          setResult('包含变量的函数表达式');
        } else {
          setResult('');
        }
      }
    } catch (parseErr) {
      setLatex(expression); // Fallback
      setIsError(true);
      setResult('');
    }
  }, [expression, isReady]);

  // 响应式画布尺寸
  useEffect(() => {
    if (!canvasContainerRef.current) return;
    const observer = new ResizeObserver(entries => {
      for (let entry of entries) {
        setDimensions({ w: entry.contentRect.width, h: entry.contentRect.height });
      }
    });
    observer.observe(canvasContainerRef.current);
    return () => observer.disconnect();
  }, []);

  // 绘制引擎
  const draw = useCallback(() => {
    if (!canvasRef.current || dimensions.w === 0 || !window.math) return;
    
    const canvas = canvasRef.current;
    const ctx = canvas.getContext('2d');
    const dpr = window.devicePixelRatio || 1;
    const w = dimensions.w;
    const h = dimensions.h;

    canvas.width = w * dpr;
    canvas.height = h * dpr;
    canvas.style.width = `${w}px`;
    canvas.style.height = `${h}px`;
    ctx.scale(dpr, dpr);

    ctx.clearRect(0, 0, w, h);

    // 坐标转换函数
    const toCanvasX = (mx) => (mx - offsetX) * scale + w / 2;
    const toCanvasY = (my) => -(my - offsetY) * scale + h / 2;
    const toMathX = (cx) => (cx - w / 2) / scale + offsetX;
    const toMathY = (cy) => -(cy - h / 2) / scale + offsetY;

    // 绘制网格
    ctx.strokeStyle = '#e2e8f0';
    ctx.lineWidth = 1;
    let step = 1;
    if (scale < 15) step = 5;
    if (scale < 5) step = 10;
    if (scale > 80) step = 0.5;

    const startX = Math.floor(toMathX(0) / step) * step;
    const endX = Math.ceil(toMathX(w) / step) * step;
    for (let x = startX; x <= endX; x += step) {
      const cx = toCanvasX(x);
      ctx.beginPath(); ctx.moveTo(cx, 0); ctx.lineTo(cx, h); ctx.stroke();
      if (x !== 0) {
        ctx.fillStyle = '#94a3b8';
        ctx.font = '10px sans-serif';
        ctx.fillText(Number(x.toPrecision(4)), cx + 4, toCanvasY(0) + 12);
      }
    }

    const startY = Math.floor(toMathY(h) / step) * step;
    const endY = Math.ceil(toMathY(0) / step) * step;
    for (let y = startY; y <= endY; y += step) {
      const cy = toCanvasY(y);
      ctx.beginPath(); ctx.moveTo(0, cy); ctx.lineTo(w, cy); ctx.stroke();
      if (y !== 0) {
        ctx.fillStyle = '#94a3b8';
        ctx.fillText(Number(y.toPrecision(4)), toCanvasX(0) + 4, cy - 4);
      }
    }

    // 绘制坐标轴
    ctx.strokeStyle = '#475569';
    ctx.lineWidth = 2;
    const y0 = toCanvasY(0);
    if (y0 >= 0 && y0 <= h) {
      ctx.beginPath(); ctx.moveTo(0, y0); ctx.lineTo(w, y0); ctx.stroke();
    }
    const x0 = toCanvasX(0);
    if (x0 >= 0 && x0 <= w) {
      ctx.beginPath(); ctx.moveTo(x0, 0); ctx.lineTo(x0, h); ctx.stroke();
    }

    // 绘制所有函数
    functions.forEach(func => {
      try {
        const node = window.math.parse(func.expr);
        const code = node.compile();

        ctx.beginPath();
        ctx.strokeStyle = func.color;
        ctx.lineWidth = 2;
        ctx.lineJoin = 'round';

        let first = true;
        for (let cx = 0; cx <= w; cx += 2) {
          const mx = toMathX(cx);
          try {
            const my = code.evaluate({ x: mx, e: Math.E, pi: Math.PI });
            if (typeof my === 'number' && !isNaN(my)) {
              const cy = toCanvasY(my);
              // 防止跨越渐近线绘制（例如tan(x)）
              if (cy < -2000 || cy > h + 2000) {
                first = true;
                continue;
              }
              if (first) { ctx.moveTo(cx, cy); first = false; }
              else { ctx.lineTo(cx, cy); }
            } else {
              first = true;
            }
          } catch (e) {
            first = true;
          }
        }
        ctx.stroke();
      } catch (e) { }
    });
  }, [dimensions, offsetX, offsetY, scale, functions]);

  useEffect(() => {
    draw();
  }, [draw]);

  // 画布交互事件
  const handleWheel = (e) => {
    e.preventDefault();
    const zoomFactor = e.deltaY > 0 ? 0.9 : 1.1;
    setScale(prev => Math.max(1, Math.min(prev * zoomFactor, 500)));
  };

  const handlePointerDown = (e) => {
    setIsDragging(true);
    setDragStart({ x: e.clientX, y: e.clientY });
  };

  const handlePointerMove = (e) => {
    if (!isDragging) return;
    const dx = e.clientX - dragStart.x;
    const dy = e.clientY - dragStart.y;
    setOffsetX(prev => prev - dx / scale);
    setOffsetY(prev => prev + dy / scale);
    setDragStart({ x: e.clientX, y: e.clientY });
  };

  const handlePointerUp = () => setIsDragging(false);

  // 键盘与操作逻辑
  const handleKey = (key) => {
    if (key === 'C') {
      setExpression('');
      setResult('');
    } else if (key === 'DEL') {
      setExpression(prev => prev.slice(0, -1));
    } else if (key === '=') {
      // 触发强制计算
    } else if (key === 'Plot') {
      if (expression && !isError) {
        setFunctions(prev => {
          // 避免重复绘制
          if (prev.find(f => f.expr === expression)) return prev;
          const color = COLORS[prev.length % COLORS.length];
          return [...prev, { id: Date.now(), expr: expression, color }];
        });
        if (window.innerWidth < 768) setMobileTab('graph');
      }
    } else if (['sin', 'cos', 'tan', 'asin', 'acos', 'atan', 'sinh', 'cosh', 'tanh', 'sqrt', 'log', 'ln', 'abs'].includes(key)) {
      if (key === 'ln') {
        setExpression(prev => prev + 'log('); // Math.js 使用 log(x) 作为自然对数
      } else if (key === 'log') {
        setExpression(prev => prev + 'log10('); // Math.js 使用 log10(x) 作为以10为底
      } else {
        setExpression(prev => prev + key + '(');
      }
    } else if (key === '!') {
      setExpression(prev => prev + '!');
    } else if (key === 'x^2') {
      setExpression(prev => prev + '^2');
    } else if (key === 'x^y') {
      setExpression(prev => prev + '^');
    } else {
      setExpression(prev => prev + key);
    }
  };

  const removeFunction = (id) => {
    setFunctions(prev => prev.filter(f => f.id !== id));
  };

  const resetView = () => {
    setOffsetX(0);
    setOffsetY(0);
    setScale(40);
  };

  const btnClass = "flex items-center justify-center w-full h-full min-h-[2.5rem] sm:min-h-[2.75rem] md:min-h-[3rem] rounded-lg sm:rounded-xl text-sm sm:text-base font-medium transition-all active:scale-95 touch-manipulation";
  const numClass = `${btnClass} bg-white text-slate-800 shadow-sm border border-slate-100 hover:bg-slate-50`;
  const opClass = `${btnClass} bg-indigo-50 text-indigo-600 shadow-sm border border-indigo-100 hover:bg-indigo-100`;
  const funcClass = `${btnClass} bg-slate-100 text-slate-700 shadow-sm hover:bg-slate-200 text-[13px] sm:text-sm`;
  const actionClass = `${btnClass} bg-indigo-500 text-white shadow-md hover:bg-indigo-600`;

  if (!isReady) return (
    <div className="min-h-screen flex items-center justify-center bg-slate-50">
      <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-indigo-500"></div>
    </div>
  );

  return (
    <div className="flex flex-col md:flex-row h-screen w-full bg-slate-50 overflow-hidden font-sans">
      
      {/* 移动端标签页切换 */}
      <div className="md:hidden flex bg-white border-b border-slate-200 shrink-0">
        <button 
          onClick={() => setMobileTab('calc')}
          className={`flex-1 flex items-center justify-center gap-2 py-3 font-medium transition-colors ${mobileTab === 'calc' ? 'text-indigo-600 border-b-2 border-indigo-600' : 'text-slate-500'}`}
        >
          <Calculator size={18} /> 计算
        </button>
        <button 
          onClick={() => setMobileTab('graph')}
          className={`flex-1 flex items-center justify-center gap-2 py-3 font-medium transition-colors ${mobileTab === 'graph' ? 'text-indigo-600 border-b-2 border-indigo-600' : 'text-slate-500'}`}
        >
          <Activity size={18} /> 图表 {functions.length > 0 && <span className="bg-indigo-100 text-indigo-600 text-xs px-2 py-0.5 rounded-full">{functions.length}</span>}
        </button>
      </div>

      {/* 左侧计算面板 */}
      <div className={`w-full md:w-[420px] flex-col border-r border-slate-200 bg-white shadow-[2px_0_10px_rgba(0,0,0,0.02)] z-10 shrink-0 ${mobileTab === 'calc' ? 'flex h-[calc(100vh-50px)] md:h-screen' : 'hidden md:flex h-screen'}`}>
        
        {/* 显示区域 */}
        <div className="flex-none p-6 border-b border-slate-100 bg-slate-50/50 flex flex-col justify-end min-h-[160px]">
          <div className="text-slate-400 text-sm mb-2 text-right h-6">
            {expression || '等待输入...'}
          </div>
          <div className="flex-1 w-full flex items-center justify-end overflow-x-auto">
             <MathDisplay tex={latex} isError={isError} />
          </div>
          <div className="text-indigo-600 font-semibold text-xl text-right h-8 mt-2">
            {result}
          </div>
        </div>

        {/* 绘图函数列表（如果在计算器页也有空间则展示） */}
        {functions.length > 0 && (
          <div className="px-4 py-3 flex gap-2 overflow-x-auto border-b border-slate-100 bg-white items-center shrink-0 hide-scrollbar">
            <span className="text-xs font-semibold text-slate-400 uppercase tracking-wider">已绘图</span>
            {functions.map(f => (
              <div key={f.id} className="flex items-center gap-1.5 px-3 py-1.5 rounded-full bg-slate-50 border border-slate-200 shadow-sm shrink-0">
                <div className="w-2.5 h-2.5 rounded-full" style={{ backgroundColor: f.color }}></div>
                <span className="text-sm font-medium">{f.expr}</span>
                <button onClick={() => removeFunction(f.id)} className="text-slate-400 hover:text-red-500 transition-colors p-0.5">
                  <Trash2 size={14} />
                </button>
              </div>
            ))}
          </div>
        )}

        {/* 高级函数折叠开关 */}
        <div className="flex justify-between items-center px-4 py-2 bg-slate-50/80 border-b border-slate-100 shrink-0">
          <span className="text-xs font-semibold text-slate-400 tracking-wider">数学键盘</span>
          <button 
            onClick={() => setShowAdvanced(!showAdvanced)} 
            className="text-xs text-indigo-600 flex items-center gap-1 hover:text-indigo-700 transition-colors bg-indigo-50 px-2 py-1 rounded-md"
          >
            {showAdvanced ? <><ChevronDown size={14}/> 收起高级函数</> : <><ChevronUp size={14}/> 展开高级函数</>}
          </button>
        </div>

        {/* 键盘区域 */}
        <div className="p-2 sm:p-4 grid grid-cols-5 gap-1.5 sm:gap-2.5 mt-auto flex-1 bg-white overflow-y-auto hide-scrollbar">
          
          {showAdvanced && (
            <>
              <button onClick={() => handleKey('asin')} className={funcClass}>asin</button>
              <button onClick={() => handleKey('acos')} className={funcClass}>acos</button>
              <button onClick={() => handleKey('atan')} className={funcClass}>atan</button>
              <button onClick={() => handleKey('log')} className={funcClass}>log</button>
              <button onClick={() => handleKey('ln')} className={funcClass}>ln</button>
              
              <button onClick={() => handleKey('sinh')} className={funcClass}>sinh</button>
              <button onClick={() => handleKey('cosh')} className={funcClass}>cosh</button>
              <button onClick={() => handleKey('tanh')} className={funcClass}>tanh</button>
              <button onClick={() => handleKey('abs')} className={funcClass}>|x|</button>
              <button onClick={() => handleKey('!')} className={funcClass}>n!</button>
            </>
          )}

          <button onClick={() => handleKey('sin')} className={funcClass}>sin</button>
          <button onClick={() => handleKey('cos')} className={funcClass}>cos</button>
          <button onClick={() => handleKey('tan')} className={funcClass}>tan</button>
          <button onClick={() => handleKey('pi')} className={funcClass}>π</button>
          <button onClick={() => handleKey('C')} className={`${funcClass} text-red-500 font-bold bg-red-50 hover:bg-red-100`}>C</button>
          
          <button onClick={() => handleKey('sqrt')} className={funcClass}>√</button>
          <button onClick={() => handleKey('^')} className={funcClass}>x^y</button>
          <button onClick={() => handleKey('x')} className={`${funcClass} font-bold text-indigo-600 bg-indigo-50`}>x</button>
          <button onClick={() => handleKey('e')} className={funcClass}>e</button>
          <button onClick={() => handleKey('DEL')} className={`${funcClass} text-slate-500`}><Delete size={18}/></button>
          
          <button onClick={() => handleKey('(')} className={funcClass}>(</button>
          <button onClick={() => handleKey(')')} className={funcClass}>)</button>
          <button onClick={() => handleKey('/')} className={opClass}>÷</button>
          <button onClick={() => handleKey('*')} className={opClass}>×</button>
          <button onClick={() => handleKey('-')} className={opClass}>−</button>
          
          <button onClick={() => handleKey('7')} className={numClass}>7</button>
          <button onClick={() => handleKey('8')} className={numClass}>8</button>
          <button onClick={() => handleKey('9')} className={numClass}>9</button>
          <button onClick={() => handleKey('+')} className={`${opClass} row-span-2`}>+</button>
          <button onClick={() => handleKey('Plot')} className={`${actionClass} row-span-2 flex flex-col gap-1 shadow-indigo-200`}>
            <Activity size={18} /> <span className="text-[11px] sm:text-sm">绘制</span>
          </button>
          
          <button onClick={() => handleKey('4')} className={numClass}>4</button>
          <button onClick={() => handleKey('5')} className={numClass}>5</button>
          <button onClick={() => handleKey('6')} className={numClass}>6</button>
          
          <button onClick={() => handleKey('1')} className={numClass}>1</button>
          <button onClick={() => handleKey('2')} className={numClass}>2</button>
          <button onClick={() => handleKey('3')} className={numClass}>3</button>
          <button onClick={() => handleKey('=')} className={`${opClass} row-span-2 font-bold text-xl sm:text-2xl`}>=</button>
          <button onClick={() => setFunctions([])} className={`${funcClass} row-span-2 text-slate-500 flex flex-col gap-1`}>
            <RefreshCcw size={16}/> <span className="text-[10px] sm:text-xs">清空</span>
          </button>
          
          <button onClick={() => handleKey('0')} className={`${numClass} col-span-2`}>0</button>
          <button onClick={() => handleKey('.')} className={numClass}>.</button>
        </div>
      </div>

      {/* 右侧绘图面板 */}
      <div className={`flex-1 relative bg-slate-50 ${mobileTab === 'graph' ? 'flex h-[calc(100vh-50px)] md:h-screen' : 'hidden md:flex h-screen'}`}>
        {/* 画布容器 */}
        <div 
          ref={canvasContainerRef} 
          className="w-full h-full cursor-grab active:cursor-grabbing touch-none"
          onWheel={handleWheel}
          onPointerDown={handlePointerDown}
          onPointerMove={handlePointerMove}
          onPointerUp={handlePointerUp}
          onPointerLeave={handlePointerUp}
        >
          <canvas ref={canvasRef} className="block w-full h-full" />
        </div>

        {/* 绘图控件悬浮窗 */}
        <div className="absolute top-4 right-4 flex flex-col gap-2 bg-white/90 backdrop-blur-sm p-2 rounded-xl shadow-sm border border-slate-200">
          <button onClick={() => setScale(s => Math.min(s * 1.2, 500))} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="放大">
            <ZoomIn size={20} />
          </button>
          <button onClick={() => setScale(s => Math.max(s * 0.8, 1))} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="缩小">
            <ZoomOut size={20} />
          </button>
          <div className="w-full h-px bg-slate-200 my-1"></div>
          <button onClick={resetView} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="重置视图">
            <Maximize size={20} />
          </button>
        </div>

        {functions.length === 0 && (
          <div className="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 text-slate-400 flex flex-col items-center gap-3 pointer-events-none text-center px-4">
            <Activity size={48} className="text-slate-300" opacity={0.5} />
            <p className="text-lg font-medium">还没有绘制任何函数</p>
            <p className="text-sm">在左侧输入包含变量 <span className="font-mono bg-slate-200 px-1.5 py-0.5 rounded text-slate-600">x</span> 的表达式并点击"ssName={`flex-1 relative bg-slate-50 ${mobileTab === 'graph' ? 'flex h-[calc(100vh-50px)] md:h-screen' : 'hidden md:flex h-screen'}`}>
        {/* 画布容器 */}
        <div 
          ref={canvasContainerRef} 
          className="w-full h-full cursor-grab active:cursor-grabbing touch-none"
          onWheel={handleWheel}
          onPointerDown={handlePointerDown}
          onPointerMove={handlePointerMove}
          onPointerUp={handlePointerUp}
          onPointerLeave={handlePointerUp}
        >
          <canvas ref={canvasRef} className="block w-full h-full" />
        </div>

        {/* 绘图控件悬浮窗 */}
        <div className="absolute top-4 right-4 flex flex-col gap-2 bg-white/90 backdrop-blur-sm p-2 rounded-xl shadow-sm border border-slate-200">
          <button onClick={() => setScale(s => Math.min(s * 1.2, 500))} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="放大">
            <ZoomIn size={20} />
          </button>
          <button onClick={() => setScale(s => Math.max(s * 0.8, 1))} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="缩小">
            <ZoomOut size={20} />
          </button>
          <div className="w-full h-px bg-slate-200 my-1"></div>
          <button onClick={resetView} className="p-2 text-slate-600 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors" title="重置视图">
            <Maximize size={20} />
          </button>
        </div>

        {functions.length === 0 && (
          <div className="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 text-slate-400 flex flex-col items-center gap-3 pointer-events-none text-center px-4">
            <Activity size={48} className="text-slate-300" opacity={0.5} />
            <p className="text-lg font-medium">还没有绘制任何函数</p>
            <p className="text-sm">在左侧输入包含变量 <span className="font-mono bg-slate-200 px-1.5 py-0.5 rounded text-slate-600">x</span> 的表达式并点击"
