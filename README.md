import React, { useState, useEffect, useCallback } from 'react';

// --- TIPOS DE DATOS ---
interface Option {
  id: string;
  text: string;
}

interface Challenge {
  id: number;
  title: string;
  situation: string;
  options: Option[];
  bgColor: string;
  textColor: string;
  borderColor: string;
}

interface Role {
  name: string;
  description: string;
  question: string;
  color: string;
  isVoter: boolean;
}

interface Vote {
  [roleName: string]: string | null; // Opción ID o null si no ha votado
}

interface ChallengeResult {
  challengeId: number;
  challengeTitle: string;
  votes: Vote; // Votos emitidos: { Estratega: 'A', Operador: 'B', ... }
  outcome: string; // Resultado según "resultados posibles" o estado de tiempo/consenso
  mentalModel: string;
  interpretation: string;
  deepInterpretation: string;
  mediatorNote: string;
  status: 'RESUELTO' | 'NO RESUELTO' | 'NO RESUELTO (TIEMPO AGOTADO)';
}

interface InterpretationEntry {
  challengeId: string;
  Estratega: string;
  Operador: string;
  Innovador: string;
  winningOption: string; // "Opción con más votos" del archivo
  mentalModel: string;
  interpretation: string;
  deepInterpretation: string;
}

// --- DATOS DEL JUEGO ---
const CHALLENGE_DURATION_SECONDS = 4 * 60; // 4 minutos

const challengesData: Challenge[] = [
  {
    id: 1,
    title: "Desafío 1: La Crisis de Talento",
    situation: "El 40% del equipo técnico renunció en las últimas 2 semanas. La presión aumenta y los proyectos clave están en riesgo. Deben decidir rápidamente cómo abordar esta fuga de talento antes de que afecte la operatividad.",
    options: [
      { id: 'A', text: "Ofrecer aumentos salariales del 30% para retener al personal actual y atraer nuevos talentos." },
      { id: 'B', text: "Contratar inmediatamente una empresa de outsourcing para cubrir las vacantes críticas." },
      { id: 'C', text: "Redistribuir las tareas y responsabilidades entre el personal restante, pidiendo un esfuerzo extra." },
      { id: 'D', text: "Cerrar temporalmente algunas líneas de operación para concentrar recursos en las más críticas." },
    ],
    bgColor: 'bg-sky-100',
    textColor: 'text-sky-800',
    borderColor: 'border-sky-500',
  },
  {
    id: 2,
    title: "Desafío 2: El Dilema del Producto",
    situation: "Su cliente principal (que representa el 60% de los ingresos) exige una nueva funcionalidad que se desvía significativamente de la visión original del producto. Amenaza con cancelar el contrato si no se cumplen sus demandas.",
    options: [
      { id: 'A', text: "Aceptar todos los requerimientos del cliente para asegurar su permanencia." },
      { id: 'B', text: "Negociar una solución intermedia, tratando de alinear sus necesidades con la visión del producto." },
      { id: 'C', text: "Mantener la visión original del producto, arriesgándose a perder al cliente." },
      { id: 'D', text: "Desarrollar dos versiones separadas del producto: una para el cliente y otra siguiendo la visión original." },
    ],
    bgColor: 'bg-amber-100',
    textColor: 'text-amber-800',
    borderColor: 'border-amber-500',
  },
  {
    id: 3,
    title: "Desafío 3: La Encrucijada Financiera",
    situation: "Un inversor ofrece 2 millones de dólares, una suma que salvaría a la empresa, pero exige el 70% del control accionario. Los fondos actuales se agotarán en 30 días.",
    options: [
      { id: 'A', text: "Aceptar la oferta inmediatamente para asegurar la supervivencia de la empresa." },
      { id: 'B', text: "Buscar activamente otros inversores, aunque esto implique el riesgo de quedarse sin fondos." },
      { id: 'C', text: "Reducir gastos drásticamente e intentar autofinanciarse con los ingresos actuales." },
      { id: 'D', text: "Buscar un préstamo bancario tradicional, asumiendo la deuda correspondiente." },
    ],
    bgColor: 'bg-emerald-100',
    textColor: 'text-emerald-800',
    borderColor: 'border-emerald-500',
  },
];

const rolesData: Record<string, Role> = {
  Estratega: { name: "Estratega", description: "Enfoque en visión y mercado.", question: "¿Cómo impacta esto nuestra posición competitiva a 3 años?", color: "bg-blue-200", isVoter: true },
  Operador: { name: "Operador", description: "Enfoque en procesos y eficiencia.", question: "¿Cómo optimizamos recursos y minimizamos riesgos operacionales?", color: "bg-yellow-200", isVoter: true },
  Innovador: { name: "Innovador", description: "Enfoque en creatividad y disrupción.", question: "¿Qué solución disruptiva nadie más está considerando?", color: "bg-purple-200", isVoter: true },
  Mediador: { name: "Mediador", description: "Enfoque en personas y relaciones (No vota).", question: "¿Cómo afecta esto a las personas y las relaciones?", color: "bg-gray-300", isVoter: false },
};
const votingRolesNames = Object.values(rolesData).filter(r => r.isVoter).map(r => r.name);


// Transcripción parcial de "resultados posibles.docx"
// IMPORTANTE: Se debe completar esta lista con TODAS las combinaciones (64 por desafío) para una funcionalidad completa.
const interpretationsData: InterpretationEntry[] = [
  // Desafío 1
  { challengeId: "1", Estratega: "A", Operador: "A", Innovador: "A", winningOption: "A", mentalModel: "Los problemas se resuelven con dinero", interpretation: "Enfoque transaccional de las relaciones laborales. Prioriza motivación extrínseca.", deepInterpretation: "Posible modelo mental de \"distancia jerárquica\" - las soluciones vienen de arriba o de afuera" },
  { challengeId: "1", Estratega: "A", Operador: "A", Innovador: "B", winningOption: "A", mentalModel: "Los problemas se resuelven con dinero", interpretation: "Enfoque transaccional de las relaciones laborales. Prioriza motivación extrínseca.", deepInterpretation: "Posible modelo mental de \"distancia jerárquica\" - las soluciones vienen de arriba o de afuera" },
  { challengeId: "1", Estratega: "A", Operador: "B", Innovador: "A", winningOption: "A", mentalModel: "Los problemas se resuelven con dinero", interpretation: "Enfoque transaccional de las relaciones laborales. Prioriza motivación extrínseca.", deepInterpretation: "Posible modelo mental de \"distancia jerárquica\" - las soluciones vienen de arriba o de afuera" },
  { challengeId: "1", Estratega: "B", Operador: "A", Innovador: "A", winningOption: "A", mentalModel: "Los problemas se resuelven con dinero", interpretation: "Enfoque transaccional de las relaciones laborales. Prioriza motivación extrínseca.", deepInterpretation: "Posible modelo mental de \"distancia jerárquica\" - las soluciones vienen de arriba o de afuera" },
  { challengeId: "1", Estratega: "A", Operador: "B", Innovador: "B", winningOption: "B", mentalModel: "Lo externo es más seguro/eficiente", interpretation: "Tendencia a buscar soluciones fuera en vez de fortalecer lo interno.", deepInterpretation: "Posible modelo mental de \"distancia jerárquica\" - las soluciones vienen de arriba o de afuera" }, // Asumiendo que B gana con A,B,B
  { challengeId: "1", Estratega: "A", Operador: "B", Innovador: "C", winningOption: "Empate entre A, B y C", mentalModel: "SIN RESOLUCIÓN", interpretation: "SIN RESOLUCIÓN", deepInterpretation: "SIN RESOLUCIÓN" },
  { challengeId: "1", Estratega: "B", Operador: "B", Innovador: "B", winningOption: "B", mentalModel: "Lo externo es más seguro/eficiente", interpretation: "Tendencia a buscar soluciones fuera en vez de fortalecer lo interno.", deepInterpretation: "Posible modelo mental de \"distancia jerárquica\" - las soluciones vienen de arriba o de afuera" },
  { challengeId: "1", Estratega: "C", Operador: "C", Innovador: "C", winningOption: "C", mentalModel: "Hay que exigir sacrificio en crisis", interpretation: "Prioriza resultados sobre bienestar. Modelo de \"crisis = esfuerzo extraordinario\".", deepInterpretation: "Posible modelo de \"escasez\" - hay que estirar recursos existentes" },
  { challengeId: "1", Estratega: "D", Operador: "D", Innovador: "D", winningOption: "D", mentalModel: "Mejor retirarse y reagrupar", interpretation: "Modelo de conservación de recursos. Prioriza sostenibilidad sobre resultados inmediatos.", deepInterpretation: "Posible modelo de \"prudencia estratégica\" - a veces hay que retroceder para avanzar" },

  // Desafío 2 (ejemplos)
  { challengeId: "2", Estratega: "A", Operador: "A", Innovador: "A", winningOption: "A", mentalModel: "El cliente siempre tiene la razón", interpretation: "Modelo de complacencia y aversión al conflicto. Prioriza ingresos sobre integridad conceptual.", deepInterpretation: "Posible modelo financiero a corto plazo y aversión al riesgo" },
  { challengeId: "2", Estratega: "B", Operador: "B", Innovador: "B", winningOption: "B", mentalModel: "El compromiso es siempre la mejor opción", interpretation: "Modelo de negociación distributiva. Busca equilibrio entre extremos sin cuestionar premisas.", deepInterpretation: "Posible modelo de \"equilibrio\" - todo problema tiene un punto medio óptimo" },
  { challengeId: "2", Estratega: "C", Operador: "C", Innovador: "C", winningOption: "C", mentalModel: "La integridad es innegociable", interpretation: "Modelo de principios rígidos. Prioriza coherencia y visión sobre adaptabilidad.", deepInterpretation: "Posible modelo de \"identidad fija\" - la esencia no debe cambiar" },
  { challengeId: "2", Estratega: "A", Operador: "B", Innovador: "C", winningOption: "Empate entre A, B y C", mentalModel: "SIN RESOLUCIÓN", interpretation: "SIN RESOLUCIÓN", deepInterpretation: "SIN RESOLUCIÓN" },


  // Desafío 3 (ejemplos)
  { challengeId: "3", Estratega: "A", Operador: "A", Innovador: "A", winningOption: "A", mentalModel: "Un pájaro en mano vale más que cien volando", interpretation: "Modelo de seguridad y aversión al riesgo extremo. Prioriza supervivencia sobre autonomía.", deepInterpretation: "Posible modelo de \"escasez\" - las oportunidades son limitadas" },
  { challengeId: "3", Estratega: "B", Operador: "B", Innovador: "B", winningOption: "B", mentalModel: "Siempre hay mejores opciones si se buscan", interpretation: "Modelo de abundancia y optimismo. Puede subestimar restricciones temporales.", deepInterpretation: "Posible modelo de \"opciones infinitas\" - siempre hay alternativas mejores" },
  { challengeId: "3", Estratega: "C", Operador: "C", Innovador: "C", winningOption: "C", mentalModel: "La independencia vale cualquier sacrificio", interpretation: "Modelo de autonomía y control. Prioriza libertad sobre recursos.", deepInterpretation: "Posible modelo de \"autosuficiencia\" - dependemos principalmente de nosotros mismos" },
  { challengeId: "3", Estratega: "A", Operador: "B", Innovador: "C", winningOption: "Empate entre A, B y C", mentalModel: "SIN RESOLUCIÓN", interpretation: "SIN RESOLUCIÓN", deepInterpretation: "SIN RESOLUCIÓN" },
];


// --- COMPONENTES ---

const Timer: React.FC<{ timeLeft: number }> = ({ timeLeft }) => {
  const minutes = Math.floor(timeLeft / 60);
  const seconds = timeLeft % 60;
  return (
    <div className="text-2xl font-bold my-4 p-3 bg-gray-700 text-white rounded-lg shadow-md">
      Tiempo Restante: {minutes}:{seconds < 10 ? `0${seconds}` : seconds}
    </div>
  );
};

const RoleCard: React.FC<{ role: Role; currentVote: string | null; onVote?: (optionId: string) => void; options?: Option[], challengeBgColor: string }> = ({ role, currentVote, onVote, options, challengeBgColor }) => {
  return (
    <div className={`p-4 rounded-lg shadow-lg border-2 ${role.color} ${challengeBgColor.includes('sky') ? 'border-sky-600' : challengeBgColor.includes('amber') ? 'border-amber-600' : 'border-emerald-600'}`}>
      <h3 className="text-xl font-semibold mb-2">{role.name}</h3>
      <p className="text-sm italic mb-1">{role.description}</p>
      <p className="text-sm mb-2">"{role.question}"</p>
      {role.isVoter && options && onVote && (
        <div className="space-y-2 mt-3">
          {options.map(option => (
            <button
              key={option.id}
              onClick={() => onVote(option.id)}
              disabled={!!currentVote}
              className={`w-full p-2 rounded-md text-sm transition-all
                ${currentVote === option.id ? 'bg-green-500 text-white ring-2 ring-green-700' : 'bg-white hover:bg-gray-100 text-gray-700 border border-gray-300'}
                ${!!currentVote && currentVote !== option.id ? 'opacity-60 cursor-not-allowed' : ''}
                ${!!currentVote ? '' : 'hover:shadow-md'}`}
            >
              {option.text}
            </button>
          ))}
        </div>
      )}
      {currentVote && role.isVoter && <p className="mt-3 text-sm font-semibold">Votó por: Opción {currentVote}</p>}
    </div>
  );
};

const MediatorNotes: React.FC<{ notes: string; onChange: (notes: string) => void, challengeBgColor: string }> = ({ notes, onChange, challengeBgColor }) => {
  return (
    <div className={`p-4 rounded-lg shadow-lg mt-6 ${challengeBgColor.includes('sky') ? 'bg-sky-50' : challengeBgColor.includes('amber') ? 'bg-amber-50' : 'bg-emerald-50'}`}>
      <h3 className="text-lg font-semibold mb-2 text-gray-700">Observaciones del Mediador:</h3>
      <textarea
        value={notes}
        onChange={(e) => onChange(e.target.value)}
        rows={4}
        className={`w-full p-2 border rounded-md focus:ring-2 ${challengeBgColor.includes('sky') ? 'border-sky-300 focus:ring-sky-500 focus:border-sky-500' : challengeBgColor.includes('amber') ? 'border-amber-300 focus:ring-amber-500 focus:border-amber-500' : 'border-emerald-300 focus:ring-emerald-500 focus:border-emerald-500' }`}
        placeholder="Escriba aquí las observaciones..."
      />
    </div>
  );
};

const ChallengeScreen: React.FC<{
  challenge: Challenge;
  timeLeft: number;
  votes: Vote;
  mediatorNote: string;
  onVote: (roleName: string, optionId: string) => void;
  onMediatorNoteChange: (note: string) => void;
  onTimeUp: () => void; // Callback for when time is up
}> = ({ challenge, timeLeft, votes, mediatorNote, onVote, onMediatorNoteChange }) => {
  
  const allVoted = votingRolesNames.every(roleName => !!votes[roleName]);

  useEffect(() => {
    if (allVoted) {
        // If all voted, can proceed without waiting for timer, or wait.
        // For this implementation, we wait for timer or explicit next button.
        // The game logic to advance is handled in App.tsx by onTimeUp or explicit advance.
    }
  }, [allVoted]);


  return (
    <div className={`min-h-screen p-4 md:p-8 flex flex-col items-center ${challenge.bgColor} ${challenge.textColor}`}>
      <div className={`w-full max-w-5xl p-6 rounded-xl shadow-2xl border-4 ${challenge.borderColor} ${challenge.bgColor.replace('100','200')}`}>
        <h1 className="text-3xl font-bold text-center mb-2">{challenge.title}</h1>
        <p className="text-center mb-6 text-md leading-relaxed">{challenge.situation}</p>
        
        <Timer timeLeft={timeLeft} />

        <div className="grid md:grid-cols-2 lg:grid-cols-3 gap-6 mb-6">
          {Object.values(rolesData).filter(r => r.isVoter).map(role => (
            <RoleCard 
              key={role.name} 
              role={role} 
              currentVote={votes[role.name] || null} 
              onVote={(optionId) => onVote(role.name, optionId)}
              options={challenge.options}
              challengeBgColor={challenge.bgColor}
            />
          ))}
        </div>
        
        <div className="mt-6">
             <RoleCard role={rolesData.Mediador} currentVote={null} challengeBgColor={challenge.bgColor} />
             <MediatorNotes notes={mediatorNote} onChange={onMediatorNoteChange} challengeBgColor={challenge.bgColor} />
        </div>

        {allVoted && (
            <div className="mt-6 p-4 bg-green-200 text-green-800 rounded-lg text-center font-semibold">
                ¡Todos los roles han votado! El facilitador indicará el siguiente paso o esperen a que termine el tiempo.
            </div>
        )}
      </div>
    </div>
  );
};

const ReportScreen: React.FC<{ results: ChallengeResult[], onRestart: () => void }> = ({ results, onRestart }) => {
  
  const generateReportText = () => {
    let report = "INFORME FINAL DEL JUEGO DE ROLES - STARTUPTECH\n";
    report += "==================================================\n\n";

    let consensusCount = 0;

    results.forEach(res => {
      report += `DESAFÍO ${res.challengeId}: ${res.challengeTitle}\n`;
      report += `--------------------------------------------------\n`;
      report += `Votos Emitidos:\n`;
      Object.entries(res.votes).forEach(([role, vote]) => {
        if (votingRolesNames.includes(role)) {
          const optionText = challengesData[res.challengeId-1].options.find(o => o.id === vote)?.text || 'No votó';
          report += `  - ${role}: Opción ${vote || 'N/A'} (${optionText})\n`;
        }
      });
      report += `\n`;
      report += `Resultado del Desafío: ${res.outcome}\n`;
      if (res.status === 'RESUELTO') {
        consensusCount++;
        report += `Modelo Mental Subyacente: ${res.mentalModel}\n`;
        report += `Interpretación Inmediata: ${res.interpretation}\n`;
        report += `Interpretación Profunda: ${res.deepInterpretation}\n`;
      } else if (res.status === 'NO RESUELTO (TIEMPO AGOTADO)') {
        report += `  El desafío no se resolvió porque se agotó el tiempo antes de que todos votaran o se llegara a un consenso.\n`;
      } else { // NO RESUELTO
         report += `  El desafío no se resolvió por falta de consenso en la votación.\n`;
         if (res.mentalModel !== "SIN RESOLUCIÓN" && res.mentalModel !== "N/A") { // Some tied results might still have interpretations
            report += `Modelo Mental (del patrón de votación): ${res.mentalModel}\n`;
            report += `Interpretación (del patrón de votación): ${res.interpretation}\n`;
            report += `Interpretación Profunda (del patrón de votación): ${res.deepInterpretation}\n`;
         }
      }
      report += `\nObservaciones del Mediador:\n  ${res.mediatorNote || "Sin observaciones."}\n`;
      report += `\n\n`;
    });

    report += `EVALUACIÓN GENERAL DEL DESEMPEÑO GRUPAL\n`;
    report += `--------------------------------------------------\n`;
    report += `Se alcanzó consenso en ${consensusCount} de ${results.length} desafíos.\n`;
    if (consensusCount === results.length) {
      report += `¡Excelente! El equipo logró construir consensos en todos los desafíos.\n`;
    } else if (consensusCount > 0) {
      report += `El equipo demostró capacidad de llegar a acuerdos, aunque hay oportunidades de mejora para los desafíos no resueltos.\n`;
    } else {
      report += `El equipo enfrentó dificultades para alcanzar consensos. Es una oportunidad para reflexionar sobre la comunicación y la toma de decisiones.\n`;
    }
    report += `\n==================================================\n`;
    return report;
  };

  const reportText = generateReportText();

  const downloadReport = () => {
    const element = document.createElement("a");
    const file = new Blob([reportText], {type: 'text/plain'});
    element.href = URL.createObjectURL(file);
    element.download = "informe_startuptech.txt";
    document.body.appendChild(element);
    element.click();
    document.body.removeChild(element);
  };

  return (
    <div className="min-h-screen bg-gray-100 p-4 md:p-8 flex flex-col items-center">
      <div className="w-full max-w-4xl bg-white p-6 md:p-8 rounded-xl shadow-2xl">
        <h1 className="text-3xl font-bold text-center mb-8 text-gray-800">Informe Integrado del Desempeño</h1>
        
        {results.map((res, index) => (
          <div key={index} className="mb-8 p-6 border rounded-lg bg-gray-50 shadow">
            <h2 className="text-2xl font-semibold mb-3 text-indigo-700">Desafío {res.challengeId}: {res.challengeTitle}</h2>
            <div className="mb-4">
              <h4 className="font-semibold text-gray-700">Votos:</h4>
              <ul className="list-disc list-inside ml-4 text-sm text-gray-600">
                {Object.entries(res.votes).map(([role, vote]) => {
                  if (votingRolesNames.includes(role)) {
                     const optionFullText = challengesData[res.challengeId-1].options.find(o => o.id === vote)?.text || 'No votó';
                     return <li key={role}>{role}: <strong>Opción {vote || 'N/A'}</strong> ({optionFullText})</li>;
                  }
                  return null;
                })}
              </ul>
            </div>
            <p className="mb-2 text-sm"><strong className="text-gray-700">Resultado:</strong> <span className={`font-semibold ${res.status === 'RESUELTO' ? 'text-green-600' : 'text-red-600'}`}>{res.outcome} ({res.status.replace(' (TIEMPO AGOTADO)', '')})</span></p>
            
            { (res.status === 'RESUELTO' || (res.status === 'NO RESUELTO' && res.mentalModel !== "SIN RESOLUCIÓN" && res.mentalModel !== "N/A")) && (
              <>
                <p className="mb-1 text-sm"><strong className="text-gray-700">Modelo Mental Subyacente:</strong> {res.mentalModel}</p>
                <p className="mb-1 text-sm"><strong className="text-gray-700">Interpretación Inmediata:</strong> {res.interpretation}</p>
                <p className="mb-1 text-sm"><strong className="text-gray-700">Interpretación Profunda:</strong> {res.deepInterpretation}</p>
              </>
            )}
             {res.status === 'NO RESUELTO (TIEMPO AGOTADO)' && (
                <p className="mb-1 text-sm text-orange-600">El desafío no se resolvió porque se agotó el tiempo.</p>
            )}

            <p className="mt-3 text-sm"><strong className="text-gray-700">Observaciones del Mediador:</strong></p>
            <p className="text-sm bg-gray-100 p-2 rounded border border-gray-200 whitespace-pre-wrap">{res.mediatorNote || "Sin observaciones."}</p>
          </div>
        ))}

        <div className="mt-8 p-6 border rounded-lg bg-indigo-50 shadow">
            <h3 className="text-xl font-semibold mb-3 text-indigo-800">Evaluación General del Desempeño Grupal</h3>
            <p className="text-gray-700">
                Se alcanzó consenso en <strong>{results.filter(r => r.status === 'RESUELTO').length}</strong> de <strong>{results.length}</strong> desafíos.
            </p>
        </div>

        <div className="mt-8 text-center space-x-4">
            <button 
                onClick={downloadReport}
                className="px-6 py-3 bg-blue-600 text-white font-semibold rounded-lg shadow-md hover:bg-blue-700 transition-colors"
            >
                Descargar Informe (.txt)
            </button>
            <button 
                onClick={onRestart}
                className="px-6 py-3 bg-gray-600 text-white font-semibold rounded-lg shadow-md hover:bg-gray-700 transition-colors"
            >
                Reiniciar Juego
            </button>
        </div>
         <div className="mt-4 text-xs text-gray-500 text-center">
            <p>Para exportar a PDF o DOCX, se requerirían bibliotecas adicionales (ej: jspdf, docx).</p>
        </div>
      </div>
    </div>
  );
};

const WelcomeScreen: React.FC<{ onStart: () => void }> = ({ onStart }) => {
  return (
    <div className="min-h-screen bg-gradient-to-br from-slate-900 to-slate-700 p-4 md:p-8 flex flex-col items-center justify-center text-white">
      <div className="w-full max-w-2xl bg-white/10 backdrop-blur-md p-8 md:p-12 rounded-xl shadow-2xl text-center">
        <h1 className="text-4xl font-bold mb-6">Bienvenido al Juego de Roles: StartupTech en Crisis</h1>
        <p className="mb-4 text-lg">
          Asumirás un rol crucial en StartupTech, una empresa que enfrenta decisiones críticas.
          Trabaja con tu equipo, debate y vota bajo presión para salvar la compañía.
        </p>
        <p className="mb-8 text-md">
          Hay 3 desafíos, cada uno con 4 minutos. El Mediador facilitará, mientras que Estratega, Operador e Innovador votarán.
          ¡Presta atención a cómo tus modelos mentales influyen en tus decisiones!
        </p>
        <button
          onClick={onStart}
          className="px-8 py-4 bg-indigo-600 text-white font-bold rounded-lg shadow-lg hover:bg-indigo-700 transition-transform hover:scale-105 text-xl"
        >
          Comenzar Juego
        </button>
      </div>
    </div>
  );
};


// --- COMPONENTE PRINCIPAL APP ---
export default function App() {
  const [currentScreen, setCurrentScreen] = useState<'WELCOME' | 'CHALLENGE' | 'REPORT'>('WELCOME');
  const [currentChallengeIndex, setCurrentChallengeIndex] = useState(0);
  const [timeLeft, setTimeLeft] = useState(CHALLENGE_DURATION_SECONDS);
  const [votes, setVotes] = useState<Record<number, Vote>>({}); // { 1: { Estratega: 'A'}, 2: ... }
  const [mediatorNotes, setMediatorNotes] = useState<Record<number, string>>({}); // { 1: "nota", 2: ... }
  const [challengeResults, setChallengeResults] = useState<ChallengeResult[]>([]);
  const [timerActive, setTimerActive] = useState(false);

  const currentChallenge = challengesData[currentChallengeIndex];
  const currentChallengeId = currentChallenge?.id;

  // Manejar la finalización de un desafío (por tiempo o por votos)
  const endChallenge = useCallback(() => {
    setTimerActive(false);
    const challengeId = currentChallenge.id;
    const currentVotesForChallenge = votes[challengeId] || {};
    
    let result: ChallengeResult;

    // Verificar si todos los roles votantes han votado
    const allVoted = votingRolesNames.every(roleName => !!currentVotesForChallenge[roleName]);

    if (allVoted) {
      const estrategaVote = currentVotesForChallenge.Estratega;
      const operadorVote = currentVotesForChallenge.Operador;
      const innovadorVote = currentVotesForChallenge.Innovador;

      const interpretationEntry = interpretationsData.find(
        entry =>
          entry.challengeId === String(challengeId) &&
          entry.Estratega === estrategaVote &&
          entry.Operador === operadorVote &&
          entry.Innovador === innovadorVote
      );

      if (interpretationEntry) {
        const isResolved = !(interpretationEntry.winningOption.includes("Empate") || interpretationEntry.winningOption.includes("SIN RESOLUCIÓN"));
        result = {
          challengeId,
          challengeTitle: currentChallenge.title,
          votes: currentVotesForChallenge,
          outcome: interpretationEntry.winningOption,
          mentalModel: interpretationEntry.mentalModel,
          interpretation: interpretationEntry.interpretation,
          deepInterpretation: interpretationEntry.deepInterpretation,
          mediatorNote: mediatorNotes[challengeId] || "",
          status: isResolved ? 'RESUELTO' : 'NO RESUELTO',
        };
      } else {
        // Combinación de votos no encontrada en interpretationsData (debería estar completa)
        // O manejar como "no resuelto" si la combinación específica no está.
        // Por ahora, si no se encuentra, es un caso no cubierto por la tabla.
        result = {
          challengeId,
          challengeTitle: currentChallenge.title,
          votes: currentVotesForChallenge,
          outcome: "Combinación de votos no definida en interpretaciones",
          mentalModel: "N/A",
          interpretation: "N/A",
          deepInterpretation: "N/A",
          mediatorNote: mediatorNotes[challengeId] || "",
          status: 'NO RESUELTO',
        };
      }
    } else {
      // No todos votaron y se acabó el tiempo
      result = {
        challengeId,
        challengeTitle: currentChallenge.title,
        votes: currentVotesForChallenge, // Guardar los votos que se hayan hecho
        outcome: "Tiempo agotado sin votación completa",
        mentalModel: "N/A",
        interpretation: "N/A",
        deepInterpretation: "N/A",
        mediatorNote: mediatorNotes[challengeId] || "",
        status: 'NO RESUELTO (TIEMPO AGOTADO)',
      };
    }

    setChallengeResults(prev => [...prev, result]);

    // Avanzar al siguiente desafío o al informe
    if (currentChallengeIndex < challengesData.length - 1) {
      setCurrentChallengeIndex(prev => prev + 1);
      setTimeLeft(CHALLENGE_DURATION_SECONDS);
      setTimerActive(true); // Iniciar timer para el siguiente desafío
    } else {
      setCurrentScreen('REPORT');
    }
  }, [currentChallenge, votes, mediatorNotes, currentChallengeIndex]);


  // Efecto para el temporizador
  useEffect(() => {
    if (currentScreen === 'CHALLENGE' && timerActive) {
      if (timeLeft <= 0) {
        endChallenge();
        return;
      }
      const timerId = setInterval(() => {
        setTimeLeft(prevTime => prevTime - 1);
      }, 1000);
      return () => clearInterval(timerId);
    }
  }, [timeLeft, currentScreen, timerActive, endChallenge]);


  const handleStartGame = () => {
    setCurrentChallengeIndex(0);
    setChallengeResults([]);
    setVotes({});
    setMediatorNotes({});
    setTimeLeft(CHALLENGE_DURATION_SECONDS);
    setCurrentScreen('CHALLENGE');
    setTimerActive(true);
  };

  const handleVote = (roleName: string, optionId: string) => {
    if (!currentChallengeId) return;
    setVotes(prev => ({
      ...prev,
      [currentChallengeId]: {
        ...prev[currentChallengeId],
        [roleName]: optionId,
      },
    }));

    // Comprobar si todos han votado después de este voto
     const updatedVotesForChallenge = { ...votes[currentChallengeId], [roleName]: optionId };
     const allHaveVoted = votingRolesNames.every(rName => !!updatedVotesForChallenge[rName]);
     if (allHaveVoted) {
        // Opcional: se podría llamar a endChallenge() aquí si se desea terminar en cuanto todos votan.
        // Por ahora, la lógica de finalización está centralizada en endChallenge(), que es llamado por el timer
        // o podría ser llamado por un botón "Finalizar Desafío" (no implementado).
        // Si se desea que el juego avance automáticamente cuando todos votan:
        // endChallenge(); 
     }
  };

  const handleMediatorNoteChange = (note: string) => {
    if (!currentChallengeId) return;
    setMediatorNotes(prev => ({
      ...prev,
      [currentChallengeId]: note,
    }));
  };
  

  if (currentScreen === 'WELCOME') {
    return <WelcomeScreen onStart={handleStartGame} />;
  }

  if (currentScreen === 'CHALLENGE' && currentChallenge) {
    return (
      <ChallengeScreen
        challenge={currentChallenge}
        timeLeft={timeLeft}
        votes={votes[currentChallenge.id] || {}}
        mediatorNote={mediatorNotes[currentChallenge.id] || ""}
        onVote={handleVote}
        onMediatorNoteChange={handleMediatorNoteChange}
        onTimeUp={endChallenge} // endChallenge se llama cuando el tiempo llega a 0
      />
    );
  }

  if (currentScreen === 'REPORT') {
    return <ReportScreen results={challengeResults} onRestart={handleStartGame} />;
  }

  return <div>Cargando juego...</div>; // Estado por defecto o de carga
}

