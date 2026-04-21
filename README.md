import React, { useEffect, useMemo, useRef, useState } from react;
import { Play, Pause, RotateCcw, Thermometer, Settings2 } from lucide-react;
import { Card, CardContent, CardHeader, CardTitle } from @componentsuicard;
import { Button } from @componentsuibutton;
import { Input } from @componentsuiinput;
import { Label } from @componentsuilabel;
import { Slider } from @componentsuislider;
import { motion } from framer-motion;

const clamp = (value, min, max) = Math.min(max, Math.max(min, value));

const DEFAULTS = {
  width 72,
  height 42,
  ambientTemp 20,
  sourcePower 220,
  sourceRadius 3.4,
  sourceSpeed 16,
  conductivity 0.16,
  cooling 0.012,
  timeStep 0.12,
  fps 30,
};

function createGrid(width, height, fill) {
  const arr = new Float32Array(width  height);
  arr.fill(fill);
  return arr;
}

function idx(x, y, width) {
  return y  width + x;
}

function colorForTemperature(temp, minT, maxT) {
  const range = Math.max(1, maxT - minT);
  const t = clamp((temp - minT)  range, 0, 1);

  const r = Math.round(255  Math.min(1, Math.max(0, 1.5  t)));
  const g = Math.round(255  Math.min(1, Math.max(0, 1.5  (1 - Math.abs(t - 0.5)  2))));
  const b = Math.round(255  Math.min(1, Math.max(0, 1.4  (1 - t))));

  return `rgb(${r}, ${g}, ${b})`;
}

function formatNumber(value, digits = 1) {
  return Number.isFinite(value)  value.toFixed(digits)  0.0;
}

export default function WeldCladdingTemperatureSite() {
  const [params, setParams] = useState(DEFAULTS);
  const [running, setRunning] = useState(false);
  const [simulationTime, setSimulationTime] = useState(0);
  const [temperature, setTemperature] = useState(() =
    createGrid(DEFAULTS.width, DEFAULTS.height, DEFAULTS.ambientTemp)
  );
  const [peakTemp, setPeakTemp] = useState(DEFAULTS.ambientTemp);
  const [currentMaxTemp, setCurrentMaxTemp] = useState(DEFAULTS.ambientTemp);
  const canvasRef = useRef(null);
  const animationRef = useRef(null);
  const accumulatorRef = useRef(0);
  const lastFrameRef = useRef(0);

  const gridSize = useMemo(() = params.width  params.height, [params.width, params.height]);

  const sourcePosition = useMemo(() = {
    const startX = 4;
    const x = (startX + simulationTime  params.sourceSpeed) % (params.width + 8) - 4;
    const y = Math.floor(params.height  2);
    return { x, y };
  }, [simulationTime, params.width, params.height, params.sourceSpeed]);

  const resetSimulation = (nextParams = params) = {
    setRunning(false);
    setSimulationTime(0);
    const nextGrid = createGrid(nextParams.width, nextParams.height, nextParams.ambientTemp);
    setTemperature(nextGrid);
    setPeakTemp(nextParams.ambientTemp);
    setCurrentMaxTemp(nextParams.ambientTemp);
    accumulatorRef.current = 0;
    lastFrameRef.current = 0;
  };

  const updateParam = (key, value) = {
    setParams((prev) = {
      const next = { ...prev, [key] value };
      return next;
    });
  };

  useEffect(() = {
    resetSimulation(params);
     eslint-disable-next-line react-hooksexhaustive-deps
  }, [params.width, params.height, params.ambientTemp]);

  useEffect(() = {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext(2d);
    const cellSize = Math.max(8, Math.floor(720  params.width));
    canvas.width = params.width  cellSize;
    canvas.height = params.height  cellSize;

    let minT = params.ambientTemp;
    let maxT = params.ambientTemp;
    for (let i = 0; i  temperature.length; i += 1) {
      const t = temperature[i];
      if (t  minT) minT = t;
      if (t  maxT) maxT = t;
    }

    for (let y = 0; y  params.height; y += 1) {
      for (let x = 0; x  params.width; x += 1) {
        const t = temperature[idx(x, y, params.width)];
        ctx.fillStyle = colorForTemperature(t, minT, maxT);
        ctx.fillRect(x  cellSize, y  cellSize, cellSize, cellSize);
      }
    }

    const srcPixelX = sourcePosition.x  cellSize;
    const srcPixelY = sourcePosition.y  cellSize;
    ctx.strokeStyle = rgba(255,255,255,0.95);
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.arc(srcPixelX, srcPixelY, Math.max(4, params.sourceRadius  cellSize), 0, Math.PI  2);
    ctx.stroke();
  }, [temperature, params, sourcePosition]);

  useEffect(() = {
    if (!running) {
      if (animationRef.current) cancelAnimationFrame(animationRef.current);
      animationRef.current = null;
      return;
    }

    const stepSimulation = () = {
      setTemperature((prev) = {
        const next = new Float32Array(gridSize);
        const w = params.width;
        const h = params.height;
        const ambient = params.ambientTemp;
        const alpha = params.conductivity;
        const cooling = params.cooling;
        const dt = params.timeStep;
        const radiusSq = params.sourceRadius  params.sourceRadius;

        let localMax = ambient;

        for (let y = 0; y  h; y += 1) {
          for (let x = 0; x  w; x += 1) {
            const i = idx(x, y, w);
            const center = prev[i];
            const left = prev[idx(Math.max(0, x - 1), y, w)];
            const right = prev[idx(Math.min(w - 1, x + 1), y, w)];
            const up = prev[idx(x, Math.max(0, y - 1), w)];
            const down = prev[idx(x, Math.min(h - 1, y + 1), w)];

            const laplacian = left + right + up + down - 4  center;

            const dx = x - sourcePosition.x;
            const dy = y - sourcePosition.y;
            const distanceSq = dx  dx + dy  dy;
            const heatInput = params.sourcePower  Math.exp(-distanceSq  (2  radiusSq));

            const coolingLoss = cooling  (center - ambient);
            const nextTemp = center + dt  (alpha  laplacian + heatInput - coolingLoss);
            next[i] = Math.max(ambient, nextTemp);
            if (next[i]  localMax) localMax = next[i];
          }
        }

        setCurrentMaxTemp(localMax);
        setPeakTemp((prevPeak) = Math.max(prevPeak, localMax));
        return next;
      });

      setSimulationTime((prev) = prev + params.timeStep);
    };

    const frameInterval = 1000  params.fps;

    const animate = (timestamp) = {
      if (!lastFrameRef.current) lastFrameRef.current = timestamp;
      const delta = timestamp - lastFrameRef.current;
      lastFrameRef.current = timestamp;
      accumulatorRef.current += delta;

      while (accumulatorRef.current = frameInterval) {
        stepSimulation();
        accumulatorRef.current -= frameInterval;
      }

      animationRef.current = requestAnimationFrame(animate);
    };

    animationRef.current = requestAnimationFrame(animate);

    return () = {
      if (animationRef.current) cancelAnimationFrame(animationRef.current);
      animationRef.current = null;
    };
  }, [running, params, gridSize, sourcePosition.x, sourcePosition.y]);

  const avgTemp = useMemo(() = {
    let sum = 0;
    for (let i = 0; i  temperature.length; i += 1) sum += temperature[i];
    return sum  Math.max(1, temperature.length);
  }, [temperature]);

  return (
    div className=min-h-screen bg-slate-950 text-slate-50
      div className=mx-auto max-w-7xl px-4 py-8 smpx-6 lgpx-8
        motion.div
          initial={{ opacity 0, y 16 }}
          animate={{ opacity 1, y 0 }}
          transition={{ duration 0.45 }}
          className=mb-8 grid gap-4 lggrid-cols-[1.35fr_0.65fr]
        
          Card className=rounded-3xl border-slate-800 bg-slate-90070 shadow-2xl
            CardHeader
              CardTitle className=flex items-center gap-3 text-2xl font-semibold
                Thermometer className=h-7 w-7 
                Модель изменения температуры при наплавке
              CardTitle
              p className=text-sm text-slate-300
                Полностью готовый интерактивный сайт тепловая карта, движущийся источник тепла и управление параметрами процесса прямо в браузере.
              p
            CardHeader
            CardContent
              div className=grid gap-4 smgrid-cols-3
                div className=rounded-2xl bg-slate-80070 p-4
                  div className=text-sm text-slate-400Время моделированияdiv
                  div className=mt-2 text-2xl font-bold{formatNumber(simulationTime, 2)} сdiv
                div
                div className=rounded-2xl bg-slate-80070 p-4
                  div className=text-sm text-slate-400Текущий максимумdiv
                  div className=mt-2 text-2xl font-bold{formatNumber(currentMaxTemp, 1)} °Cdiv
                div
                div className=rounded-2xl bg-slate-80070 p-4
                  div className=text-sm text-slate-400Пиковая температураdiv
                  div className=mt-2 text-2xl font-bold{formatNumber(peakTemp, 1)} °Cdiv
                div
              div
            CardContent
          Card

          Card className=rounded-3xl border-slate-800 bg-slate-90070 shadow-2xl
            CardHeader
              CardTitle className=flex items-center gap-3 text-xl font-semibold
                Settings2 className=h-6 w-6 
                Управление
              CardTitle
            CardHeader
            CardContent className=space-y-3
              div className=grid grid-cols-3 gap-2
                Button
                  className=rounded-2xl
                  onClick={() = setRunning((prev) = !prev)}
                
                  {running  Pause className=mr-2 h-4 w-4   Play className=mr-2 h-4 w-4 }
                  {running  Пауза  Старт}
                Button
                Button
                  variant=secondary
                  className=rounded-2xl
                  onClick={() = resetSimulation()}
                
                  RotateCcw className=mr-2 h-4 w-4 
                  Сброс
                Button
                Button
                  variant=outline
                  className=rounded-2xl border-slate-700 bg-slate-900
                  onClick={() = {
                    setParams(DEFAULTS);
                    resetSimulation(DEFAULTS);
                  }}
                
                  По умолчанию
                Button
              div

              div className=rounded-2xl bg-slate-80070 p-4 text-sm text-slate-300
                Белый круг показывает положение источника тепла. Чем ярче цвет карты, тем выше температура в этой области.
              div
            CardContent
          Card
        motion.div

        div className=grid gap-6 lggrid-cols-[1.3fr_0.7fr]
          motion.div
            initial={{ opacity 0, y 18 }}
            animate={{ opacity 1, y 0 }}
            transition={{ duration 0.5, delay 0.08 }}
          
            Card className=rounded-3xl border-slate-800 bg-slate-90070 shadow-2xl
              CardHeader
                CardTitle className=text-xl font-semiboldТепловая картаCardTitle
              CardHeader
              CardContent
                div className=overflow-hidden rounded-3xl border border-slate-800 bg-black
                  canvas ref={canvasRef} className=block h-auto w-full 
                div

                div className=mt-4 grid gap-3 smgrid-cols-4
                  div className=rounded-2xl bg-slate-80070 p-4
                    div className=text-xs text-slate-400Средняя температураdiv
                    div className=mt-1 text-lg font-semibold{formatNumber(avgTemp, 1)} °Cdiv
                  div
                  div className=rounded-2xl bg-slate-80070 p-4
                    div className=text-xs text-slate-400Позиция источника Xdiv
                    div className=mt-1 text-lg font-semibold{formatNumber(sourcePosition.x, 1)}div
                  div
                  div className=rounded-2xl bg-slate-80070 p-4
                    div className=text-xs text-slate-400Ширина сеткиdiv
                    div className=mt-1 text-lg font-semibold{params.width}div
                  div
                  div className=rounded-2xl bg-slate-80070 p-4
                    div className=text-xs text-slate-400Высота сеткиdiv
                    div className=mt-1 text-lg font-semibold{params.height}div
                  div
                div
              CardContent
            Card
          motion.div

          motion.div
            initial={{ opacity 0, y 18 }}
            animate={{ opacity 1, y 0 }}
            transition={{ duration 0.5, delay 0.14 }}
          
            Card className=rounded-3xl border-slate-800 bg-slate-90070 shadow-2xl
              CardHeader
                CardTitle className=text-xl font-semiboldПараметры процессаCardTitle
              CardHeader
              CardContent className=space-y-6
                ParameterSlider
                  label=Температура среды
                  value={params.ambientTemp}
                  min={0}
                  max={100}
                  step={1}
                  suffix=°C
                  onChange={(value) = updateParam(ambientTemp, value)}
                
                ParameterSlider
                  label=Мощность источника
                  value={params.sourcePower}
                  min={20}
                  max={500}
                  step={1}
                  suffix=усл. ед.
                  onChange={(value) = updateParam(sourcePower, value)}
                
                ParameterSlider
                  label=Радиус источника
                  value={params.sourceRadius}
                  min={1}
                  max={8}
                  step={0.1}
                  suffix=клетки
                  onChange={(value) = updateParam(sourceRadius, value)}
                
                ParameterSlider
                  label=Скорость наплавки
                  value={params.sourceSpeed}
                  min={2}
                  max={30}
                  step={0.5}
                  suffix=клетс
                  onChange={(value) = updateParam(sourceSpeed, value)}
                
                ParameterSlider
                  label=Теплопроводность
                  value={params.conductivity}
                  min={0.01}
                  max={0.5}
                  step={0.01}
                  suffix=
                  onChange={(value) = updateParam(conductivity, value)}
                
                ParameterSlider
                  label=Охлаждение
                  value={params.cooling}
                  min={0}
                  max={0.08}
                  step={0.001}
                  suffix=
                  onChange={(value) = updateParam(cooling, value)}
                
                ParameterSlider
                  label=Шаг по времени
                  value={params.timeStep}
                  min={0.02}
                  max={0.4}
                  step={0.01}
                  suffix=с
                  onChange={(value) = updateParam(timeStep, value)}
                

                div className=grid grid-cols-2 gap-4 pt-2
                  NumberField
                    id=grid-width
                    label=Ширина сетки
                    value={params.width}
                    min={24}
                    max={140}
                    onChange={(value) = updateParam(width, clamp(value, 24, 140))}
                  
                  NumberField
                    id=grid-height
                    label=Высота сетки
                    value={params.height}
                    min={20}
                    max={90}
                    onChange={(value) = updateParam(height, clamp(value, 20, 90))}
                  
                div
              CardContent
            Card
          motion.div
        div
      div
    div
  );
}

function ParameterSlider({ label, value, min, max, step, suffix, onChange }) {
  return (
    div className=space-y-2
      div className=flex items-center justify-between gap-4
        Label className=text-sm font-medium text-slate-200{label}Label
        span className=text-sm text-slate-400
          {formatNumber(value, step  1  2  0)} {suffix}
        span
      div
      Slider
        value={[value]}
        min={min}
        max={max}
        step={step}
        onValueChange={(values) = onChange(values[0])}
      
    div
  );
}

function NumberField({ id, label, value, min, max, onChange }) {
  return (
    div className=space-y-2
      Label htmlFor={id} className=text-sm font-medium text-slate-200
        {label}
      Label
      Input
        id={id}
        type=number
        min={min}
        max={max}
        value={value}
        className=rounded-2xl border-slate-700 bg-slate-950 text-slate-50
        onChange={(e) = {
          const raw = Number(e.target.value);
          if (!Number.isNaN(raw)) onChange(raw);
        }}
      
    div
  );
}
