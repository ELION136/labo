import React, { useState, useEffect, useRef } from 'react';
import { AlertCircle, Play, RotateCcw, Settings, FileText } from 'lucide-react';

const LadderFallSimulator = () => {
  const canvasRef = useRef(null);
  const [config, setConfig] = useState({
    height: 3,
    ladderType: 'apoyada',
    angle: 75,
    surface: 'seco',
    ppe: 'ninguno',
    workerState: 'atento'
  });
  
  const [simulating, setSimulating] = useState(false);
  const [results, setResults] = useState(null);
  const [animationFrame, setAnimationFrame] = useState(0);

  // Factores de riesgo
  const factors = {
    ppe: { ninguno: 1, casco: 0.7, arnes: 0.3, 'arnes+linea': 0.1 },
    surface: { seco: 1, mojado: 1.3, inestable: 1.5 },
    ladderType: { apoyada: 1, tijera: 1.1, extensible: 1.2, defectuosa: 1.8 },
    workerState: { atento: 1, fatigado: 1.3, 'mal_apoyo': 1.6 }
  };

  // Clase para el personaje articulado
  class Worker {
    constructor(x, y) {
      this.x = x;
      this.y = y;
      this.rotation = 0;
      this.velocity = { x: 0, y: 0 };
      this.angularVelocity = 0;
      this.onLadder = true;
      
      // Partes del cuerpo
      this.head = { radius: 8 };
      this.torso = { width: 12, height: 25 };
      this.arms = { left: 0, right: 0 }; // ángulos
      this.legs = { left: 0, right: 0 }; // ángulos
    }

    fall() {
      this.onLadder = false;
      this.velocity.y = 0;
      this.velocity.x = Math.random() * 2 - 1;
      this.angularVelocity = (Math.random() - 0.5) * 0.3;
    }

    update(gravity = 0.5) {
      if (!this.onLadder) {
        this.velocity.y += gravity;
        this.x += this.velocity.x;
        this.y += this.velocity.y;
        this.rotation += this.angularVelocity;

        // Animar extremidades durante la caída
        this.arms.left = Math.sin(this.rotation * 2) * 45;
        this.arms.right = Math.sin(this.rotation * 2 + Math.PI) * 45;
        this.legs.left = Math.sin(this.rotation * 3) * 30;
        this.legs.right = Math.sin(this.rotation * 3 + Math.PI) * 30;

        // Colisión con el suelo
        if (this.y > 450) {
          this.y = 450;
          this.velocity.y *= -0.3; // Rebote
          this.velocity.x *= 0.8;
          this.angularVelocity *= 0.7;
          
          if (Math.abs(this.velocity.y) < 1) {
            this.velocity.y = 0;
            this.velocity.x = 0;
            this.angularVelocity = 0;
          }
        }
      }
    }

    draw(ctx) {
      ctx.save();
      ctx.translate(this.x, this.y);
      ctx.rotate(this.rotation);

      // Casco (si lo tiene)
      if (config.ppe === 'casco' || config.ppe === 'arnes' || config.ppe === 'arnes+linea') {
        ctx.fillStyle = '#FFA500';
        ctx.beginPath();
        ctx.arc(0, -20, 10, Math.PI, 0);
        ctx.fill();
      }

      // Cabeza
      ctx.fillStyle = '#FFD4A3';
      ctx.beginPath();
      ctx.arc(0, -20, this.head.radius, 0, Math.PI * 2);
      ctx.fill();
      ctx.strokeStyle = '#333';
      ctx.lineWidth = 1;
      ctx.stroke();

      // Torso
      ctx.fillStyle = '#4A90E2';
      ctx.fillRect(
        -this.torso.width / 2,
        -10,
        this.torso.width,
        this.torso.height
      );
      ctx.strokeRect(
        -this.torso.width / 2,
        -10,
        this.torso.width,
        this.torso.height
      );

      // Arnés (si lo tiene)
      if (config.ppe === 'arnes' || config.ppe === 'arnes+linea') {
        ctx.strokeStyle = '#FF6B6B';
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.moveTo(-6, -5);
        ctx.lineTo(6, -5);
        ctx.moveTo(0, -5);
        ctx.lineTo(0, 15);
        ctx.stroke();
      }

      // Brazo izquierdo
      ctx.save();
      ctx.translate(-6, -5);
      ctx.rotate((this.arms.left * Math.PI) / 180);
      ctx.strokeStyle = '#FFD4A3';
      ctx.lineWidth = 3;
      ctx.beginPath();
      ctx.moveTo(0, 0);
      ctx.lineTo(0, 15);
      ctx.stroke();
      ctx.restore();

      // Brazo derecho
      ctx.save();
      ctx.translate(6, -5);
      ctx.rotate((this.arms.right * Math.PI) / 180);
      ctx.strokeStyle = '#FFD4A3';
      ctx.lineWidth = 3;
      ctx.beginPath();
      ctx.moveTo(0, 0);
      ctx.lineTo(0, 15);
      ctx.stroke();
      ctx.restore();

      // Pierna izquierda
      ctx.save();
      ctx.translate(-4, 15);
      ctx.rotate((this.legs.left * Math.PI) / 180);
      ctx.strokeStyle = '#2C3E50';
      ctx.lineWidth = 3;
      ctx.beginPath();
      ctx.moveTo(0, 0);
      ctx.lineTo(0, 18);
      ctx.stroke();
      ctx.restore();

      // Pierna derecha
      ctx.save();
      ctx.translate(4, 15);
      ctx.rotate((this.legs.right * Math.PI) / 180);
      ctx.strokeStyle = '#2C3E50';
      ctx.lineWidth = 3;
      ctx.beginPath();
      ctx.moveTo(0, 0);
      ctx.lineTo(0, 18);
      ctx.stroke();
      ctx.restore();

      ctx.restore();
    }
  }

  // Clase para la escalera
  class Ladder {
    constructor(x, y, height, angle, type) {
      this.x = x;
      this.y = y;
      this.height = height * 30; // Convertir metros a píxeles
      this.angle = angle;
      this.type = type;
      this.width = 40;
      this.stepCount = Math.floor(height * 3);
    }

    draw(ctx) {
      ctx.save();
      ctx.translate(this.x, this.y);
      
      const angleRad = (this.angle * Math.PI) / 180;
      const ladderLength = this.height / Math.sin(angleRad);

      if (this.type === 'tijera') {
        this.drawStepLadder(ctx, ladderLength);
      } else {
        this.drawLeaningLadder(ctx, ladderLength, angleRad);
      }

      ctx.restore();
    }

    drawLeaningLadder(ctx, length, angleRad) {
      ctx.rotate(-angleRad);

      // Laterales
      ctx.strokeStyle = '#8B4513';
      ctx.lineWidth = 6;
      ctx.beginPath();
      ctx.moveTo(-this.width / 2, 0);
      ctx.lineTo(-this.width / 2, -length);
      ctx.stroke();

      ctx.beginPath();
      ctx.moveTo(this.width / 2, 0);
      ctx.lineTo(this.width / 2, -length);
      ctx.stroke();

      // Peldaños
      ctx.strokeStyle = '#A0522D';
      ctx.lineWidth = 4;
      const stepSpacing = length / this.stepCount;
      
      for (let i = 0; i < this.stepCount; i++) {
        const y = -i * stepSpacing;
        ctx.beginPath();
        ctx.moveTo(-this.width / 2, y);
        ctx.lineTo(this.width / 2, y);
        ctx.stroke();
      }

      // Sombras en laterales
      ctx.strokeStyle = '#654321';
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(-this.width / 2 + 2, 0);
      ctx.lineTo(-this.width / 2 + 2, -length);
      ctx.stroke();
    }

    drawStepLadder(ctx, length) {
      const halfLength = length / 2;

      // Lado izquierdo
      ctx.save();
      ctx.rotate(-Math.PI / 4);
      ctx.strokeStyle = '#8B4513';
      ctx.lineWidth = 6;
      ctx.beginPath();
      ctx.moveTo(-this.width / 2, 0);
      ctx.lineTo(-this.width / 2, -halfLength);
      ctx.stroke();
      ctx.beginPath();
      ctx.moveTo(this.width / 2, 0);
      ctx.lineTo(this.width / 2, -halfLength);
      ctx.stroke();
      ctx.restore();

      // Lado derecho
      ctx.save();
      ctx.rotate(Math.PI / 4);
      ctx.strokeStyle = '#8B4513';
      ctx.lineWidth = 6;
      ctx.beginPath();
      ctx.moveTo(-this.width / 2, 0);
      ctx.lineTo(-this.width / 2, -halfLength);
      ctx.stroke();
      ctx.beginPath();
      ctx.moveTo(this.width / 2, 0);
      ctx.lineTo(this.width / 2, -halfLength);
      ctx.stroke();
      ctx.restore();
    }
  }

  // Calcular riesgo
  const calculateRisk = () => {
    const heightFactor = config.height;
    const ppeFactor = factors.ppe[config.ppe];
    const surfaceFactor = factors.surface[config.surface];
    const ladderFactor = factors.ladderType[config.ladderType];
    const stateFactor = factors.workerState[config.workerState];

    const totalRisk = heightFactor * ppeFactor * surfaceFactor * ladderFactor * stateFactor;
    
    // Energía de impacto E = mgh (asumiendo masa de 70kg)
    const energy = 70 * 9.8 * config.height;

    let severity, injuries, causes, recommendations;

    if (totalRisk < 2) {
      severity = 'Leve';
      injuries = ['Contusiones menores', 'Esguinces leves', 'Hematomas'];
    } else if (totalRisk < 4) {
      severity = 'Moderada';
      injuries = ['Fracturas simples', 'Esguinces graves', 'Contusiones severas', 'Posible conmoción'];
    } else if (totalRisk < 7) {
      severity = 'Grave';
      injuries = ['Fracturas múltiples', 'Traumatismo craneoencefálico', 'Lesiones vertebrales', 'Hemorragias internas'];
    } else {
      severity = 'Fatal';
      injuries = ['Politraumatismo severo', 'Traumatismo craneoencefálico grave', 'Lesiones incompatibles con la vida'];
    }

    causes = [];
    if (config.ppe === 'ninguno') causes.push('Falta de equipo de protección personal');
    if (config.surface !== 'seco') causes.push('Superficie de apoyo inadecuada');
    if (config.ladderType === 'defectuosa') causes.push('Escalera en mal estado');
    if (config.workerState !== 'atento') causes.push('Estado del trabajador comprometido');
    if (config.angle < 70 || config.angle > 80) causes.push('Ángulo de apoyo incorrecto');

    recommendations = [
      'Uso obligatorio de arnés con línea de vida',
      'Inspección previa de la escalera',
      'Verificar estabilidad del suelo',
      'Mantener tres puntos de apoyo',
      'No trabajar en estado de fatiga',
      'Utilizar escalera apropiada para la altura'
    ];

    return {
      totalRisk: totalRisk.toFixed(2),
      severity,
      injuries,
      energy: energy.toFixed(0),
      causes,
      recommendations
    };
  };

  // Simulación
  const startSimulation = () => {
    setSimulating(true);
    setResults(null);
    setAnimationFrame(0);

    const canvas = canvasRef.current;
    const ctx = canvas.getContext('2d');
    
    const ladder = new Ladder(300, 450, config.height, config.angle, config.ladderType);
    const worker = new Worker(300, 450 - config.height * 25);

    let frame = 0;
    let climbingUp = true;
    let falling = false;

    const animate = () => {
      ctx.clearRect(0, 0, canvas.width, canvas.height);

      // Fondo
      ctx.fillStyle = '#E8F4F8';
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      // Suelo
      ctx.fillStyle = config.surface === 'mojado' ? '#5DADE2' : 
                      config.surface === 'inestable' ? '#D4A574' : '#7D8B8C';
      ctx.fillRect(0, 450, canvas.width, 50);

      // Textura del suelo
      for (let i = 0; i < canvas.width; i += 20) {
        ctx.strokeStyle = 'rgba(0,0,0,0.1)';
        ctx.beginPath();
        ctx.moveTo(i, 450);
        ctx.lineTo(i, 500);
        ctx.stroke();
      }

      ladder.draw(ctx);

      // Fase de subida
      if (climbingUp && frame < 60) {
        worker.y = 450 - config.height * 25 - frame * 0.5;
        worker.arms.left = Math.sin(frame * 0.3) * 20;
        worker.arms.right = Math.sin(frame * 0.3 + Math.PI) * 20;
        worker.legs.left = Math.sin(frame * 0.3) * 15;
        worker.legs.right = Math.sin(frame * 0.3 + Math.PI) * 15;
      } else if (climbingUp) {
        climbingUp = false;
        falling = true;
        worker.fall();
      }

      // Fase de caída
      if (falling) {
        worker.update();
        
        // Finalizar cuando se detenga
        if (worker.y >= 450 && Math.abs(worker.velocity.y) < 0.5) {
          falling = false;
          setSimulating(false);
          setResults(calculateRisk());
          return;
        }
      }

      worker.draw(ctx);

      // Indicadores
      ctx.fillStyle = '#333';
      ctx.font = '14px Arial';
      ctx.fillText(`Altura: ${config.height}m`, 10, 30);
      ctx.fillText(`Ángulo: ${config.angle}°`, 10, 50);
      ctx.fillText(`EPP: ${config.ppe}`, 10, 70);

      frame++;
      setAnimationFrame(frame);
      requestAnimationFrame(animate);
    };

    animate();
  };

  useEffect(() => {
    const canvas = canvasRef.current;
    if (canvas) {
      const ctx = canvas.getContext('2d');
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      
      // Vista inicial
      ctx.fillStyle = '#E8F4F8';
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = '#7D8B8C';
      ctx.fillRect(0, 450, canvas.width, 50);
      
      const ladder = new Ladder(300, 450, config.height, config.angle, config.ladderType);
      ladder.draw(ctx);
      
      const worker = new Worker(300, 450 - config.height * 25);
      worker.draw(ctx);
    }
  }, [config]);

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-gray-100 p-6">
      <div className="max-w-7xl mx-auto">
        <div className="bg-white rounded-lg shadow-2xl overflow-hidden">
          {/* Header */}
          <div className="bg-gradient-to-r from-red-600 to-orange-600 p-6 text-white">
            <div className="flex items-center gap-3">
              <AlertCircle size={32} />
              <div>
                <h1 className="text-3xl font-bold">Simulador de Caídas en Trabajo en Alturas</h1>
                <p className="text-red-100">Herramienta educativa de seguridad industrial</p>
              </div>
            </div>
          </div>

          <div className="grid md:grid-cols-3 gap-6 p-6">
            {/* Panel de configuración */}
            <div className="space-y-4">
              <div className="bg-blue-50 p-4 rounded-lg border-l-4 border-blue-600">
                <div className="flex items-center gap-2 mb-4">
                  <Settings size={20} />
                  <h2 className="font-bold text-lg">Configuración</h2>
                </div>

                <div className="space-y-3">
                  <div>
                    <label className="block text-sm font-semibold mb-1">
                      Altura de trabajo: {config.height}m
                    </label>
                    <input
                      type="range"
                      min="1"
                      max="8"
                      step="0.5"
                      value={config.height}
                      onChange={(e) => setConfig({...config, height: parseFloat(e.target.value)})}
                      className="w-full"
                    />
                  </div>

                  <div>
                    <label className="block text-sm font-semibold mb-1">Tipo de escalera</label>
                    <select
                      value={config.ladderType}
                      onChange={(e) => setConfig({...config, ladderType: e.target.value})}
                      className="w-full border rounded px-3 py-2"
                    >
                      <option value="apoyada">Apoyada</option>
                      <option value="tijera">Tijera</option>
                      <option value="extensible">Extensible</option>
                      <option value="defectuosa">Defectuosa</option>
                    </select>
                  </div>

                  <div>
                    <label className="block text-sm font-semibold mb-1">
                      Ángulo: {config.angle}°
                    </label>
                    <input
                      type="range"
                      min="45"
                      max="85"
                      value={config.angle}
                      onChange={(e) => setConfig({...config, angle: parseInt(e.target.value)})}
                      className="w-full"
                    />
                  </div>

                  <div>
                    <label className="block text-sm font-semibold mb-1">Condición del suelo</label>
                    <select
                      value={config.surface}
                      onChange={(e) => setConfig({...config, surface: e.target.value})}
                      className="w-full border rounded px-3 py-2"
                    >
                      <option value="seco">Seco</option>
                      <option value="mojado">Mojado</option>
                      <option value="inestable">Inestable</option>
                    </select>
                  </div>

                  <div>
                    <label className="block text-sm font-semibold mb-1">Equipo de protección</label>
                    <select
                      value={config.ppe}
                      onChange={(e) => setConfig({...config, ppe: e.target.value})}
                      className="w-full border rounded px-3 py-2"
                    >
                      <option value="ninguno">Sin equipo</option>
                      <option value="casco">Solo casco</option>
                      <option value="arnes">Arnés</option>
                      <option value="arnes+linea">Arnés + Línea de vida</option>
                    </select>
                  </div>

                  <div>
                    <label className="block text-sm font-semibold mb-1">Estado del trabajador</label>
                    <select
                      value={config.workerState}
                      onChange={(e) => setConfig({...config, workerState: e.target.value})}
                      className="w-full border rounded px-3 py-2"
                    >
                      <option value="atento">Atento</option>
                      <option value="fatigado">Fatigado</option>
                      <option value="mal_apoyo">Mal apoyo</option>
                    </select>
                  </div>
                </div>

                <button
                  onClick={startSimulation}
                  disabled={simulating}
                  className="w-full mt-4 bg-red-600 hover:bg-red-700 disabled:bg-gray-400 text-white font-bold py-3 rounded-lg flex items-center justify-center gap-2 transition"
                >
                  {simulating ? (
                    <>Simulando...</>
                  ) : (
                    <>
                      <Play size={20} />
                      Iniciar Simulación
                    </>
                  )}
                </button>
              </div>
            </div>

            {/* Área de simulación */}
            <div className="md:col-span-2 space-y-4">
              <div className="bg-white border-2 border-gray-300 rounded-lg overflow-hidden">
                <canvas
                  ref={canvasRef}
                  width={600}
                  height={500}
                  className="w-full"
                />
              </div>

              {/* Resultados */}
              {results && (
                <div className="bg-gradient-to-br from-red-50 to-orange-50 p-6 rounded-lg border-2 border-red-200">
                  <div className="flex items-center gap-2 mb-4">
                    <FileText size={24} className="text-red-600" />
                    <h2 className="text-xl font-bold">Resultados del Accidente</h2>
                  </div>

                  <div className="grid md:grid-cols-2 gap-4">
                    <div className="bg-white p-4 rounded-lg">
                      <h3 className="font-bold text-red-600 mb-2">Nivel de Lesión</h3>
                      <p className="text-2xl font-bold">{results.severity}</p>
                      <p className="text-sm text-gray-600 mt-1">Índice de riesgo: {results.totalRisk}</p>
                    </div>

                    <div className="bg-white p-4 rounded-lg">
                      <h3 className="font-bold text-orange-600 mb-2">Energía de Impacto</h3>
                      <p className="text-2xl font-bold">{results.energy} J</p>
                      <p className="text-sm text-gray-600 mt-1">E = mgh (m=70kg)</p>
                    </div>
                  </div>

                  <div className="mt-4 bg-white p-4 rounded-lg">
                    <h3 className="font-bold text-red-600 mb-2">Lesiones Probables</h3>
                    <ul className="list-disc list-inside space-y-1">
                      {results.injuries.map((injury, i) => (
                        <li key={i} className="text-sm">{injury}</li>
                      ))}
                    </ul>
                  </div>

                  <div className="mt-4 bg-white p-4 rounded-lg">
                    <h3 className="font-bold text-orange-600 mb-2">Causas Probables</h3>
                    <ul className="list-disc list-inside space-y-1">
                      {results.causes.map((cause, i) => (
                        <li key={i} className="text-sm">{cause}</li>
                      ))}
                    </ul>
                  </div>

                  <div className="mt-4 bg-white p-4 rounded-lg">
                    <h3 className="font-bold text-green-600 mb-2">Recomendaciones Preventivas</h3>
                    <ul className="list-disc list-inside space-y-1">
                      {results.recommendations.map((rec, i) => (
                        <li key={i} className="text-sm">{rec}</li>
                      ))}
                    </ul>
                  </div>

                  <button
                    onClick={() => {
                      setResults(null);
                      setAnimationFrame(0);
                    }}
                    className="w-full mt-4 bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 rounded-lg flex items-center justify-center gap-2 transition"
                  >
                    <RotateCcw size={20} />
                    Nueva Simulación
                  </button>
                </div>
              )}
            </div>
          </div>
        </div>

        {/* Información adicional */}
        <div className="mt-6 bg-yellow-50 border-l-4 border-yellow-600 p-4 rounded-lg">
          <h3 className="font-bold text-yellow-800 mb-2">⚠️ Advertencia Educativa</h3>
          <p className="text-sm text-yellow-800">
            Este simulador tiene fines exclusivamente educativos. Los cálculos son aproximados y no sustituyen 
            una evaluación profesional de riesgos. Siempre consulte con un profesional en seguridad industrial 
            antes de realizar trabajos en altura.
          </p>
        </div>
      </div>
    </div>
  );
};

export default LadderFallSimulator;