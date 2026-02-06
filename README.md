# semar-flight-planner
flight planner
<!DOCTYPE html>

<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <meta name="apple-mobile-web-app-title" content="SEMAR FP">
    <meta name="theme-color" content="#3b82f6">
    <meta name="description" content="Sistema de Planificación de Vuelo SEMAR">

```
<title>SEMAR Flight Planner - PWA</title>

<!-- PWA Manifest -->
<link rel="manifest" href="manifest.json">

<!-- Apple Touch Icons -->
<link rel="apple-touch-icon" href="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'%3E%3Crect fill='%233b82f6' width='100' height='100'/%3E%3Ctext x='50' y='65' font-size='60' text-anchor='middle' fill='white'%3E⚓%3C/text%3E%3C/svg%3E">

<script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
<script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
<script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
<script src="https://cdn.tailwindcss.com"></script>
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

<style>
    @import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap');
    
    * {
        -webkit-tap-highlight-color: transparent;
    }
    
    body {
        font-family: 'Inter', sans-serif;
        background: linear-gradient(135deg, #0f172a 0%, #1e293b 100%);
        margin: 0;
        padding: 0;
        overflow-x: hidden;
        -webkit-user-select: none;
        user-select: none;
    }
    
    .glass-panel {
        background: rgba(255, 255, 255, 0.05);
        backdrop-filter: blur(10px);
        border: 1px solid rgba(255, 255, 255, 0.1);
    }
    
    .aviation-grid {
        background-image: 
            linear-gradient(rgba(59, 130, 246, 0.1) 1px, transparent 1px),
            linear-gradient(90deg, rgba(59, 130, 246, 0.1) 1px, transparent 1px);
        background-size: 20px 20px;
    }
    
    .metar-display {
        font-family: 'Courier New', monospace;
        background: rgba(0, 0, 0, 0.3);
        border-left: 3px solid #3b82f6;
    }
    
    #map {
        height: 400px;
        border-radius: 12px;
        z-index: 1;
    }
    
    .loading-spinner {
        border: 3px solid rgba(255, 255, 255, 0.1);
        border-radius: 50%;
        border-top-color: #3b82f6;
        animation: spin 1s ease-in-out infinite;
    }
    
    @keyframes spin {
        to { transform: rotate(360deg); }
    }
    
    .slide-in {
        animation: slideIn 0.3s ease-out;
    }
    
    @keyframes slideIn {
        from {
            opacity: 0;
            transform: translateY(20px);
        }
        to {
            opacity: 1;
            transform: translateY(0);
        }
    }
    
    .stat-card {
        background: linear-gradient(135deg, rgba(59, 130, 246, 0.1) 0%, rgba(147, 51, 234, 0.1) 100%);
        border: 1px solid rgba(59, 130, 246, 0.3);
        transition: transform 0.2s;
    }
    
    .stat-card:active {
        transform: scale(0.95);
    }
    
    .btn-primary {
        background: linear-gradient(135deg, #3b82f6 0%, #2563eb 100%);
        transition: all 0.2s;
    }
    
    .btn-primary:active {
        transform: scale(0.95);
    }
    
    input, select, button {
        -webkit-appearance: none;
        appearance: none;
    }
</style>
```

</head>
<body>
    <div id="root"></div>

```
<script type="text/babel">
    const { useState, useEffect, useCallback, useRef } = React;

    // ==================== BASE DE DATOS LOCAL ====================
    class LocalDatabase {
        constructor() {
            this.dbName = 'semar_flight_db';
            this.init();
        }

        init() {
            // Crear estructura si no existe
            if (!localStorage.getItem(this.dbName)) {
                const initialDB = {
                    users: [
                        { id: 1, username: 'piloto1', password: 'semar2024', name: 'Cap. Juan Pérez', rank: 'Capitán', role: 'pilot' },
                        { id: 2, username: 'piloto2', password: 'semar2024', name: 'Tte. María García', rank: 'Teniente', role: 'pilot' },
                        { id: 3, username: 'admin', password: 'admin2024', name: 'Comandante Silva', rank: 'Comandante', role: 'admin' }
                    ],
                    flights: [],
                    aircraft: this.getAircraftDB(),
                    airports: this.getAirportsDB()
                };
                localStorage.setItem(this.dbName, JSON.stringify(initialDB));
            }
        }

        getDB() {
            return JSON.parse(localStorage.getItem(this.dbName));
        }

        saveDB(db) {
            localStorage.setItem(this.dbName, JSON.stringify(db));
        }

        // Usuarios
        authenticateUser(username, password) {
            const db = this.getDB();
            return db.users.find(u => u.username === username && u.password === password);
        }

        // Vuelos
        saveFlight(flightData, weatherData, analysis, user) {
            const db = this.getDB();
            const flight = {
                id: Date.now(),
                userId: user.id,
                userName: user.name,
                flightData,
                weatherData,
                analysis,
                createdAt: new Date().toISOString(),
                status: analysis.decision
            };
            db.flights.unshift(flight);
            // Mantener solo últimos 100 vuelos
            if (db.flights.length > 100) {
                db.flights = db.flights.slice(0, 100);
            }
            this.saveDB(db);
            return flight;
        }

        getFlights(userId = null, limit = 20) {
            const db = this.getDB();
            let flights = db.flights;
            if (userId) {
                flights = flights.filter(f => f.userId === userId);
            }
            return flights.slice(0, limit);
        }

        getFlightById(id) {
            const db = this.getDB();
            return db.flights.find(f => f.id === id);
        }

        deleteFlightById(id) {
            const db = this.getDB();
            db.flights = db.flights.filter(f => f.id !== id);
            this.saveDB(db);
        }

        // Estadísticas
        getStats(userId = null) {
            const db = this.getDB();
            let flights = db.flights;
            if (userId) {
                flights = flights.filter(f => f.userId === userId);
            }

            const total = flights.length;
            const goFlights = flights.filter(f => f.analysis.decision === 'GO').length;
            const noGoFlights = flights.filter(f => f.analysis.decision === 'NO-GO').length;
            
            // Aeronaves más usadas
            const aircraftCount = {};
            flights.forEach(f => {
                const ac = f.flightData.aircraft;
                aircraftCount[ac] = (aircraftCount[ac] || 0) + 1;
            });

            // Rutas más frecuentes
            const routeCount = {};
            flights.forEach(f => {
                const route = `${f.flightData.departure}-${f.flightData.destination}`;
                routeCount[route] = (routeCount[route] || 0) + 1;
            });

            return {
                total,
                goFlights,
                noGoFlights,
                goPercentage: total > 0 ? ((goFlights / total) * 100).toFixed(1) : 0,
                topAircraft: Object.entries(aircraftCount).sort((a, b) => b[1] - a[1]).slice(0, 5),
                topRoutes: Object.entries(routeCount).sort((a, b) => b[1] - a[1]).slice(0, 5),
                recentFlights: flights.slice(0, 5)
            };
        }

        getAircraftDB() {
            return {
                helicopters: {
                    'UH-60 Black Hawk': {
                        type: 'Helicóptero', category: 'Medio', maxSpeed: 183, cruiseSpeed: 150,
                        maxAltitude: 19000, range: 360, endurance: 2.3, maxCrosswind: 35,
                        minVisibility: 800, ceiling: 200, icingCapability: 'Limited', maxWeight: 9980
                    },
                    'AS365 Panther': {
                        type: 'Helicóptero', category: 'Medio', maxSpeed: 165, cruiseSpeed: 140,
                        maxAltitude: 15000, range: 450, endurance: 3.5, maxCrosswind: 30,
                        minVisibility: 1000, ceiling: 300, icingCapability: 'None', maxWeight: 4300
                    },
                    'Bolkow Bo-105': {
                        type: 'Helicóptero', category: 'Ligero', maxSpeed: 150, cruiseSpeed: 130,
                        maxAltitude: 17000, range: 350, endurance: 2.8, maxCrosswind: 25,
                        minVisibility: 1200, ceiling: 400, icingCapability: 'None', maxWeight: 2500
                    }
                },
                fixedWing: {
                    'CASA C-295M': {
                        type: 'Avión', category: 'Transporte Táctico', maxSpeed: 260, cruiseSpeed: 230,
                        maxAltitude: 25000, range: 1300, endurance: 5.5, maxCrosswind: 40,
                        minVisibility: 550, ceiling: 200, icingCapability: 'Full', maxWeight: 23200
                    },
                    'Beechcraft King Air 350ER': {
                        type: 'Avión', category: 'Patrulla/Reconocimiento', maxSpeed: 312, cruiseSpeed: 280,
                        maxAltitude: 35000, range: 2800, endurance: 8, maxCrosswind: 35,
                        minVisibility: 400, ceiling: 200, icingCapability: 'Full', maxWeight: 7300
                    }
                },
                uavs: {
                    'Hermes 900': {
                        type: 'UAV', category: 'MALE', maxSpeed: 120, cruiseSpeed: 70,
                        maxAltitude: 30000, range: 200, endurance: 30, maxCrosswind: 25,
                        minVisibility: 3000, ceiling: 500, icingCapability: 'Limited', maxWeight: 1180
                    }
                }
            };
        }

        getAirportsDB() {
            return {
                'MMMX': { name: 'Aeropuerto Internacional de la Ciudad de México', lat: 19.4363, lon: -99.0721, city: 'Ciudad de México' },
                'MMGL': { name: 'Aeropuerto Internacional de Guadalajara', lat: 20.5218, lon: -103.3106, city: 'Guadalajara' },
                'MMUN': { name: 'Aeropuerto Internacional de Cancún', lat: 21.0365, lon: -86.8770, city: 'Cancún' },
                'MMTM': { name: 'Aeropuerto Internacional de Tampico', lat: 22.2964, lon: -97.8659, city: 'Tampico' },
                'MMMY': { name: 'Aeropuerto Internacional de Mérida', lat: 20.9370, lon: -89.6577, city: 'Mérida' },
                'MMTJ': { name: 'Aeropuerto Internacional de Tijuana', lat: 32.5411, lon: -116.9700, city: 'Tijuana' },
                'MMMD': { name: 'Aeropuerto Internacional de Manzanillo', lat: 19.1448, lon: -104.5586, city: 'Manzanillo' },
                'MMVR': { name: 'Aeropuerto Internacional de Veracruz', lat: 19.1459, lon: -96.1873, city: 'Veracruz' }
            };
        }
    }

    const db = new LocalDatabase();

    // ==================== SERVICIO METEOROLÓGICO ====================
    class WeatherAPIService {
        constructor() {
            this.cache = new Map();
            this.cacheDuration = 15 * 60 * 1000;
        }

        async getMETAR(icao) {
            const cached = this.getFromCache(`metar_${icao}`);
            if (cached) return cached;

            try {
                const url = `https://aviationweather.gov/api/data/metar?ids=${icao}&format=json`;
                const response = await fetch(url);
                if (!response.ok) throw new Error('NOAA failed');
                const data = await response.json();
                if (!data || data.length === 0) throw new Error('No data');

                const result = {
                    source: 'NOAA AWC',
                    icao: icao,
                    raw: data[0].rawOb || `METAR ${icao} DATA UNAVAILABLE`,
                    timestamp: data[0].obsTime || new Date().toISOString(),
                    conditions: {
                        wind: { speed: data[0].wspd || 10, direction: data[0].wdir || 270, gust: data[0].wgst || 0 },
                        visibility: data[0].visib || 9999,
                        ceiling: data[0].ceil || 5000,
                        temperature: data[0].temp || 20,
                        dewpoint: data[0].dwpt || 10,
                        qnh: data[0].altim ? data[0].altim * 33.8639 : 1013,
                        crosswind: Math.round((data[0].wspd || 10) * 0.5),
                        phenomena: data[0].wxString ? [data[0].wxString] : []
                    }
                };
                this.setCache(`metar_${icao}`, result);
                return result;
            } catch (error) {
                return this.getMockMETAR(icao);
            }
        }

        getMockMETAR(icao) {
            return {
                source: 'SIMULADO',
                icao: icao,
                raw: `METAR ${icao} 061500Z 27012KT 9999 FEW025 20/10 Q1015 NOSIG`,
                timestamp: new Date().toISOString(),
                conditions: {
                    wind: { speed: 12, gust: 0, direction: 270 },
                    visibility: 9999,
                    ceiling: 2500,
                    temperature: 20,
                    dewpoint: 10,
                    qnh: 1015,
                    crosswind: 8,
                    phenomena: []
                }
            };
        }

        getFromCache(key) {
            const cached = this.cache.get(key);
            if (cached && Date.now() - cached.timestamp < this.cacheDuration) {
                return cached.data;
            }
            return null;
        }

        setCache(key, data) {
            this.cache.set(key, { data, timestamp: Date.now() });
        }
    }

    const weatherAPI = new WeatherAPIService();

    // ==================== MISIONES ====================
    const MISSION_TYPES = {
        SAR: { name: 'Búsqueda y Rescate (SAR)', minVisibility: 3000, maxWinds: 35, requiresLoiter: true, loiterTime: 2 },
        PATROL: { name: 'Patrullaje Marítimo', minVisibility: 5000, maxWinds: 40, requiresLoiter: true, loiterTime: 4 },
        TRANSPORT: { name: 'Transporte', minVisibility: 1600, maxWinds: 45, requiresLoiter: false, loiterTime: 0 },
        RECON: { name: 'Reconocimiento', minVisibility: 8000, maxWinds: 30, requiresLoiter: true, loiterTime: 3 },
        TRAINING: { name: 'Entrenamiento', minVisibility: 5000, maxWinds: 25, requiresLoiter: false, loiterTime: 1 }
    };

    // ==================== COMPONENTE LOGIN ====================
    function LoginScreen({ onLogin }) {
        const [username, setUsername] = useState('');
        const [password, setPassword] = useState('');
        const [error, setError] = useState('');

        const handleLogin = (e) => {
            e.preventDefault();
            const user = db.authenticateUser(username, password);
            if (user) {
                onLogin(user);
            } else {
                setError('Usuario o contraseña incorrectos');
            }
        };

        return (
            <div className="min-h-screen flex items-center justify-center p-4 aviation-grid">
                <div className="glass-panel rounded-2xl p-8 w-full max-w-md slide-in">
                    <div className="text-center mb-8">
                        <div className="w-20 h-20 bg-gradient-to-br from-blue-500 to-blue-700 rounded-xl flex items-center justify-center text-white font-bold text-3xl mx-auto mb-4">
                            ⚓
                        </div>
                        <h1 className="text-3xl font-bold text-white mb-2">SEMAR Flight Planner</h1>
                        <p className="text-blue-300">Secretaría de Marina - Armada de México</p>
                    </div>

                    <form onSubmit={handleLogin} className="space-y-4">
                        <div>
                            <label className="block text-blue-300 mb-2 font-semibold">Usuario</label>
                            <input
                                type="text"
                                value={username}
                                onChange={(e) => setUsername(e.target.value)}
                                className="w-full bg-gray-800 text-white border border-gray-600 rounded-lg p-3 focus:border-blue-500 outline-none"
                                placeholder="piloto1"
                                autoComplete="username"
                            />
                        </div>

                        <div>
                            <label className="block text-blue-300 mb-2 font-semibold">Contraseña</label>
                            <input
                                type="password"
                                value={password}
                                onChange={(e) => setPassword(e.target.value)}
                                className="w-full bg-gray-800 text-white border border-gray-600 rounded-lg p-3 focus:border-blue-500 outline-none"
                                placeholder="••••••••"
                                autoComplete="current-password"
                            />
                        </div>

                        {error && (
                            <div className="bg-red-900/30 border border-red-500 rounded-lg p-3 text-red-300 text-sm">
                                {error}
                            </div>
                        )}

                        <button
                            type="submit"
                            className="w-full btn-primary text-white py-3 rounded-lg font-semibold"
                        >
                            Iniciar Sesión
                        </button>
                    </form>

                    <div className="mt-6 p-4 bg-blue-900/20 rounded-lg">
                        <p className="text-xs text-blue-300 mb-2">Usuarios de prueba:</p>
                        <div className="text-xs text-gray-400 space-y-1">
                            <div>👤 piloto1 / semar2024</div>
                            <div>👤 piloto2 / semar2024</div>
                            <div>👑 admin / admin2024</div>
                        </div>
                    </div>
                </div>
            </div>
        );
    }

    // ==================== COMPONENTE DASHBOARD ====================
    function Dashboard({ user, onNewFlight }) {
        const [stats, setStats] = useState(null);
        const [recentFlights, setRecentFlights] = useState([]);

        useEffect(() => {
            loadStats();
        }, []);

        const loadStats = () => {
            const userStats = db.getStats(user.role === 'admin' ? null : user.id);
            setStats(userStats);
            setRecentFlights(userStats.recentFlights);
        };

        const deleteFlightHandler = (id) => {
            if (confirm('¿Eliminar este plan de vuelo?')) {
                db.deleteFlightById(id);
                loadStats();
            }
        };

        if (!stats) {
            return (
                <div className="min-h-screen flex items-center justify-center">
                    <div className="loading-spinner w-16 h-16"></div>
                </div>
            );
        }

        return (
            <div className="min-h-screen p-4 md:p-8 aviation-grid">
                <div className="max-w-7xl mx-auto">
                    {/* Header */}
                    <div className="glass-panel rounded-2xl p-6 mb-6 flex justify-between items-center">
                        <div>
                            <h1 className="text-2xl font-bold text-white">Dashboard SEMAR</h1>
                            <p className="text-blue-300">{user.name} • {user.rank}</p>
                        </div>
                        <button
                            onClick={onNewFlight}
                            className="btn-primary text-white px-6 py-3 rounded-lg font-semibold"
                        >
                            ✈️ Nuevo Plan de Vuelo
                        </button>
                    </div>

                    {/* Stats Cards */}
                    <div className="grid md:grid-cols-4 gap-4 mb-6">
                        <div className="stat-card rounded-xl p-6">
                            <div className="text-blue-300 text-sm mb-2">Total de Vuelos</div>
                            <div className="text-4xl font-bold text-white">{stats.total}</div>
                        </div>
                        <div className="stat-card rounded-xl p-6">
                            <div className="text-green-300 text-sm mb-2">Vuelos GO</div>
                            <div className="text-4xl font-bold text-green-400">{stats.goFlights}</div>
                        </div>
                        <div className="stat-card rounded-xl p-6">
                            <div className="text-red-300 text-sm mb-2">Vuelos NO-GO</div>
                            <div className="text-4xl font-bold text-red-400">{stats.noGoFlights}</div>
                        </div>
                        <div className="stat-card rounded-xl p-6">
                            <div className="text-yellow-300 text-sm mb-2">% Aprobación</div>
                            <div className="text-4xl font-bold text-yellow-400">{stats.goPercentage}%</div>
                        </div>
                    </div>

                    {/* Charts */}
                    <div className="grid md:grid-cols-2 gap-6 mb-6">
                        {/* Top Aircraft */}
                        <div className="glass-panel rounded-2xl p-6">
                            <h3 className="text-xl font-bold text-white mb-4">Aeronaves Más Usadas</h3>
                            <div className="space-y-3">
                                {stats.topAircraft.map(([aircraft, count], idx) => (
                                    <div key={idx} className="flex items-center justify-between">
                                        <span className="text-white text-sm">{aircraft}</span>
                                        <div className="flex items-center">
                                            <div className="w-32 bg-gray-700 rounded-full h-2 mr-3">
                                                <div 
                                                    className="bg-blue-500 h-2 rounded-full" 
                                                    style={{ width: `${(count / stats.total) * 100}%` }}
                                                ></div>
                                            </div>
                                            <span className="text-blue-300 text-sm font-semibold">{count}</span>
                                        </div>
                                    </div>
                                ))}
                            </div>
                        </div>

                        {/* Top Routes */}
                        <div className="glass-panel rounded-2xl p-6">
                            <h3 className="text-xl font-bold text-white mb-4">Rutas Más Frecuentes</h3>
                            <div className="space-y-3">
                                {stats.topRoutes.map(([route, count], idx) => (
                                    <div key={idx} className="flex items-center justify-between">
                                        <span className="text-white text-sm font-mono">{route}</span>
                                        <div className="flex items-center">
                                            <div className="w-32 bg-gray-700 rounded-full h-2 mr-3">
                                                <div 
                                                    className="bg-purple-500 h-2 rounded-full" 
                                                    style={{ width: `${(count / stats.total) * 100}%` }}
                                                ></div>
                                            </div>
                                            <span className="text-purple-300 text-sm font-semibold">{count}</span>
                                        </div>
                                    </div>
                                ))}
                            </div>
                        </div>
                    </div>

                    {/* Recent Flights */}
                    <div className="glass-panel rounded-2xl p-6">
                        <h3 className="text-xl font-bold text-white mb-4">Vuelos Recientes</h3>
                        <div className="overflow-x-auto">
                            <table className="w-full text-white">
                                <thead>
                                    <tr className="border-b border-gray-700">
                                        <th className="text-left py-3 px-4 text-blue-300">Fecha</th>
                                        <th className="text-left py-3 px-4 text-blue-300">Piloto</th>
                                        <th className="text-left py-3 px-4 text-blue-300">Aeronave</th>
                                        <th className="text-left py-3 px-4 text-blue-300">Ruta</th>
                                        <th className="text-left py-3 px-4 text-blue-300">Estado</th>
                                        <th className="text-left py-3 px-4 text-blue-300">Acciones</th>
                                    </tr>
                                </thead>
                                <tbody>
                                    {recentFlights.map((flight) => (
                                        <tr key={flight.id} className="border-b border-gray-800 hover:bg-white/5">
                                            <td className="py-3 px-4 text-sm">
                                                {new Date(flight.createdAt).toLocaleDateString('es-MX')}
                                            </td>
                                            <td className="py-3 px-4 text-sm">{flight.userName}</td>
                                            <td className="py-3 px-4 text-sm">{flight.flightData.aircraft}</td>
                                            <td className="py-3 px-4 text-sm font-mono">
                                                {flight.flightData.departure} → {flight.flightData.destination}
                                            </td>
                                            <td className="py-3 px-4">
                                                <span className={`px-3 py-1 rounded-full text-xs font-semibold ${
                                                    flight.status === 'GO' 
                                                        ? 'bg-green-900/30 text-green-400' 
                                                        : 'bg-red-900/30 text-red-400'
                                                }`}>
                                                    {flight.status}
                                                </span>
                                            </td>
                                            <td className="py-3 px-4">
                                                <button
                                                    onClick={() => deleteFlightHandler(flight.id)}
                                                    className="text-red-400 hover:text-red-300 text-sm"
                                                >
                                                    🗑️
                                                </button>
                                            </td>
                                        </tr>
                                    ))}
                                </tbody>
                            </table>
                        </div>
                    </div>
                </div>
            </div>
        );
    }

    // ==================== COMPONENTE FLIGHT PLANNER ====================
    function FlightPlanner({ user, onBackToDashboard }) {
        const [step, setStep] = useState(1);
        const [flightData, setFlightData] = useState({
            aircraft: '',
            mission: '',
            departure: '',
            destination: '',
            alternate1: '',
            alternate2: '',
            route: '',
            altitude: '',
            etd: '',
            loiterTime: 0
        });
        const [weatherData, setWeatherData] = useState(null);
        const [analysis, setAnalysis] = useState(null);
        const [loading, setLoading] = useState(false);

        const allAircraft = {
            ...db.getDB().aircraft.helicopters,
            ...db.getDB().aircraft.fixedWing,
            ...db.getDB().aircraft.uavs
        };

        const handleInputChange = (field, value) => {
            setFlightData(prev => ({ ...prev, [field]: value }));
        };

        const getAircraftData = () => {
            return allAircraft[flightData.aircraft] || null;
        };

        const fetchWeatherData = async () => {
            setLoading(true);
            try {
                const [departure, destination, alt1, alt2] = await Promise.all([
                    weatherAPI.getMETAR(flightData.departure),
                    weatherAPI.getMETAR(flightData.destination),
                    weatherAPI.getMETAR(flightData.alternate1),
                    weatherAPI.getMETAR(flightData.alternate2)
                ]);

                const weatherPackage = {
                    departure, destination,
                    alternate1: alt1,
                    alternate2: alt2,
                    timestamp: new Date().toISOString()
                };

                setWeatherData(weatherPackage);
                performAnalysis(weatherPackage);
            } catch (error) {
                alert('Error al obtener datos meteorológicos');
            } finally {
                setLoading(false);
            }
        };

        const performAnalysis = (weather) => {
            const aircraft = getAircraftData();
            const limitations = [];
            const warnings = [];
            let goNoGo = 'GO';
            let score = 100;

            if (weather.departure.conditions.crosswind > aircraft.maxCrosswind) {
                limitations.push({
                    severity: 'CRITICAL',
                    factor: 'Viento Cruzado en Salida',
                    value: `${weather.departure.conditions.crosswind} kt`,
                    limit: `${aircraft.maxCrosswind} kt`,
                    impact: 'Excede límites de la aeronave'
                });
                goNoGo = 'NO-GO';
                score -= 30;
            }

            if (weather.destination.conditions.visibility < aircraft.minVisibility) {
                limitations.push({
                    severity: 'CRITICAL',
                    factor: 'Visibilidad en Destino',
                    value: `${weather.destination.conditions.visibility} m`,
                    limit: `${aircraft.minVisibility} m`,
                    impact: 'Por debajo de mínimos'
                });
                goNoGo = 'NO-GO';
                score -= 40;
            }

            if (weather.destination.conditions.ceiling < aircraft.ceiling) {
                limitations.push({
                    severity: 'CRITICAL',
                    factor: 'Techo en Destino',
                    value: `${weather.destination.conditions.ceiling} ft`,
                    limit: `${aircraft.ceiling} ft`,
                    impact: 'Techo por debajo de mínimos'
                });
                goNoGo = 'NO-GO';
                score -= 40;
            }

            const distance = 150;
            const groundSpeed = aircraft.cruiseSpeed - 10;
            const flightTime = (distance / groundSpeed).toFixed(1);
            const totalTime = parseFloat(flightTime) + parseFloat(flightData.loiterTime || 0);

            setAnalysis({
                decision: goNoGo,
                score: Math.max(0, score),
                limitations,
                warnings,
                flightTime,
                totalTime,
                fuelRequired: (totalTime * 150).toFixed(0),
                recommendations: generateRecommendations(goNoGo, limitations),
                timestamp: new Date().toISOString()
            });
        };

        const generateRecommendations = (decision, limitations) => {
            const recs = [];
            if (decision === 'NO-GO') {
                recs.push('⛔ VUELO NO RECOMENDADO - Factores críticos presentes');
                recs.push('Considerar postponer hasta mejorar condiciones');
            } else {
                recs.push('✅ CONDICIONES FAVORABLES PARA EL VUELO');
            }
            recs.push('Verificar NOTAMs antes del despegue');
            recs.push('Mantener comunicación con control de tráfico');
            return recs;
        };

        const saveFlight = () => {
            db.saveFlight(flightData, weatherData, analysis, user);
            alert('✅ Plan de vuelo guardado exitosamente');
            onBackToDashboard();
        };

        return (
            <div className="min-h-screen p-4 md:p-8 aviation-grid">
                <div className="max-w-5xl mx-auto">
                    {/* Header */}
                    <div className="glass-panel rounded-2xl p-4 mb-6 flex justify-between items-center">
                        <button
                            onClick={onBackToDashboard}
                            className="text-blue-400 hover:text-blue-300"
                        >
                            ← Dashboard
                        </button>
                        <h1 className="text-xl font-bold text-white">Nuevo Plan de Vuelo</h1>
                        <div className="w-20"></div>
                    </div>

                    {/* Stepper */}
                    <div className="glass-panel rounded-xl p-4 mb-6">
                        <div className="flex items-center justify-between">
                            {[
                                { num: 1, label: 'Aeronave' },
                                { num: 2, label: 'Ruta' },
                                { num: 3, label: 'Análisis' }
                            ].map((s, idx) => (
                                <React.Fragment key={s.num}>
                                    <div className="flex flex-col items-center">
                                        <div className={`w-10 h-10 rounded-full flex items-center justify-center font-bold
                                            ${step >= s.num ? 'bg-blue-500 text-white' : 'bg-gray-600 text-gray-400'}`}>
                                            {s.num}
                                        </div>
                                        <div className="text-white text-xs mt-2">{s.label}</div>
                                    </div>
                                    {idx < 2 && (
                                        <div className={`flex-1 h-1 mx-2 ${step > s.num ? 'bg-blue-500' : 'bg-gray-600'}`} />
                                    )}
                                </React.Fragment>
                            ))}
                        </div>
                    </div>

                    {/* Step 1 */}
                    {step === 1 && (
                        <div className="glass-panel rounded-2xl p-6 slide-in">
                            <h2 className="text-xl font-bold text-white mb-4">Selección de Aeronave y Misión</h2>
                            
                            <div className="space-y-4 mb-6">
                                <div>
                                    <label className="block text-blue-300 mb-2">Aeronave</label>
                                    <select 
                                        className="w-full bg-gray-800 text-white border border-gray-600 rounded-lg p-3"
                                        value={flightData.aircraft}
                                        onChange={(e) => handleInputChange('aircraft', e.target.value)}
                                    >
                                        <option value="">Seleccionar...</option>
                                        <optgroup label="Helicópteros">
                                            {Object.keys(db.getDB().aircraft.helicopters).map(ac => (
                                                <option key={ac} value={ac}>{ac}</option>
                                            ))}
                                        </optgroup>
                                        <optgroup label="Aviones">
                                            {Object.keys(db.getDB().aircraft.fixedWing).map(ac => (
                                                <option key={ac} value={ac}>{ac}</option>
                                            ))}
                                        </optgroup>
                                        <optgroup label="UAVs">
                                            {Object.keys(db.getDB().aircraft.uavs).map(ac => (
                                                <option key={ac} value={ac}>{ac}</option>
                                            ))}
                                        </optgroup>
                                    </select>
                                </div>

                                <div>
                                    <label className="block text-blue-300 mb-2">Misión</label>
                                    <select 
                                        className="w-full bg-gray-800 text-white border border-gray-600 rounded-lg p-3"
                                        value={flightData.mission}
                                        onChange={(e) => handleInputChange('mission', e.target.value)}
                                    >
                                        <option value="">Seleccionar...</option>
                                        {Object.entries(MISSION_TYPES).map(([key, mission]) => (
                                            <option key={key} value={key}>{mission.name}</option>
                                        ))}
                                    </select>
                                </div>
                            </div>

                            <button
                                onClick={() => setStep(2)}
                                disabled={!flightData.aircraft || !flightData.mission}
                                className="w-full btn-primary disabled:opacity-50 text-white py-3 rounded-lg font-semibold"
                            >
                                Continuar →
                            </button>
                        </div>
                    )}

                    {/* Step 2 */}
                    {step === 2 && (
                        <div className="glass-panel rounded-2xl p-6 slide-in">
                            <h2 className="text-xl font-bold text-white mb-4">Plan de Vuelo</h2>
                            
                            <div className="grid grid-cols-2 gap-4 mb-6">
                                <div>
                                    <label className="block text-blue-300 mb-2 text-sm">Salida (ICAO)</label>
                                    <input
                                        type="text"
                                        maxLength="4"
                                        className="w-full bg-gray-800 text-white border border-gray-600 rounded-lg p-3 uppercase"
                                        placeholder="MMMX"
                                        value={flightData.departure}
                                        onChange={(e) => handleInputChange('departure', e.target.value.toUpperCase())}
                                    />
                                </div>
                                <div>
                                    <label className="block text-blue-300 mb-2 text-sm">Destino (ICAO)</label>
                                    <input
                                        type="text"
                                        maxLength="4"
                                        className="w-full bg-gray-800 text-white border border-gray-600 rounded-lg p-3 uppercase"
                                        placeholder="MMGL"
                                        value={flightData.destination}
                                        onChange={(e) => handleInputChange('destination', e.target.value.toUpperCase())}
                                    />
                                </div>
                                <div>
                                    <label className="block text-blue-300 mb-2 text-sm">Alterno 1</label>
                                    <input
                                        type="text"
                                        maxLength="4"
                                        className="w-full bg-gray-800 text-white border border-gray-600 rounded-lg p-3 uppercase"
                                        placeholder="MMUN"
                                        value={flightData.alternate1}
                                        onChange={(e) => handleInputChange('alternate1', e.target.value.toUpperCase())}
                                    />
                                </div>
                                <div>
                                    <label className="block text-blue-300 mb-2 text-sm">Alterno 2</label>
                                    <input
                                        type="text"
                                        maxLength="4"
                                        className="w-full bg-gray-800 text-white border border-gray-600 rounded-lg p-3 uppercase"
                                        placeholder="MMTM"
                                        value={flightData.alternate2}
                                        onChange={(e) => handleInputChange('alternate2', e.target.value.toUpperCase())}
                                    />
                                </div>
                                <div>
                                    <label className="block text-blue-300 mb-2 text-sm">Altitud</label>
                                    <input
                                        type="text"
                                        className="w-full bg-gray-800 text-white border border-gray-600 rounded-lg p-3"
                                        placeholder="FL085"
                                        value={flightData.altitude}
                                        onChange={(e) => handleInputChange('altitude', e.target.value)}
                                    />
                                </div>
                                <div>
                                    <label className="block text-blue-300 mb-2 text-sm">Hora Despegue</label>
                                    <input
                                        type="datetime-local"
                                        className="w-full bg-gray-800 text-white border border-gray-600 rounded-lg p-3"
                                        value={flightData.etd}
                                        onChange={(e) => handleInputChange('etd', e.target.value)}
                                    />
                                </div>
                            </div>

                            <div className="flex gap-3">
                                <button
                                    onClick={() => setStep(1)}
                                    className="flex-1 bg-gray-600 text-white py-3 rounded-lg font-semibold"
                                >
                                    ← Atrás
                                </button>
                                <button
                                    onClick={() => {
                                        setStep(3);
                                        fetchWeatherData();
                                    }}
                                    disabled={!flightData.departure || !flightData.destination}
                                    className="flex-1 btn-primary disabled:opacity-50 text-white py-3 rounded-lg font-semibold"
                                >
                                    Analizar →
                                </button>
                            </div>
                        </div>
                    )}

                    {/* Step 3 */}
                    {step === 3 && (
                        <div className="space-y-4">
                            {loading ? (
                                <div className="glass-panel rounded-2xl p-12 text-center">
                                    <div className="loading-spinner w-16 h-16 mx-auto mb-4"></div>
                                    <div className="text-white">Analizando condiciones...</div>
                                </div>
                            ) : weatherData && analysis ? (
                                <>
                                    <div className={`glass-panel rounded-2xl p-6 border-4 ${
                                        analysis.decision === 'GO' ? 'border-green-500' : 'border-red-500'
                                    }`}>
                                        <div className="flex items-center justify-between">
                                            <div>
                                                <h2 className="text-3xl font-bold" style={{
                                                    color: analysis.decision === 'GO' ? '#10b981' : '#ef4444'
                                                }}>
                                                    {analysis.decision === 'GO' ? '✅ GO' : '⛔ NO-GO'}
                                                </h2>
                                                <p className="text-white">Puntuación: {analysis.score}/100</p>
                                            </div>
                                            <div className="text-6xl font-bold" style={{
                                                color: analysis.decision === 'GO' ? '#10b981' : '#ef4444'
                                            }}>
                                                {analysis.score}
                                            </div>
                                        </div>
                                    </div>

                                    <div className="glass-panel rounded-2xl p-6">
                                        <h3 className="text-lg font-bold text-white mb-3">METAR</h3>
                                        <div className="space-y-2">
                                            {['departure', 'destination'].map(loc => (
                                                <div key={loc} className="metar-display p-3 rounded text-xs">
                                                    <div className="text-blue-400 mb-1">{weatherData[loc].icao}</div>
                                                    <div className="text-green-400 font-mono">{weatherData[loc].raw}</div>
                                                </div>
                                            ))}
                                        </div>
                                    </div>

                                    {analysis.limitations.length > 0 && (
                                        <div className="glass-panel rounded-2xl p-6">
                                            <h3 className="text-lg font-bold text-red-400 mb-3">Limitaciones Críticas</h3>
                                            {analysis.limitations.map((lim, idx) => (
                                                <div key={idx} className="bg-red-900/20 border-l-4 border-red-500 p-3 rounded mb-2">
                                                    <div className="font-semibold text-red-400 text-sm">{lim.factor}</div>
                                                    <div className="text-white text-xs mt-1">{lim.value} / Límite: {lim.limit}</div>
                                                </div>
                                            ))}
                                        </div>
                                    )}

                                    <div className="glass-panel rounded-2xl p-6">
                                        <h3 className="text-lg font-bold text-white mb-3">Recomendaciones</h3>
                                        <ul className="space-y-2">
                                            {analysis.recommendations.map((rec, idx) => (
                                                <li key={idx} className="text-white text-sm">• {rec}</li>
                                            ))}
                                        </ul>
                                    </div>

                                    <div className="flex gap-3">
                                        <button
                                            onClick={() => setStep(2)}
                                            className="flex-1 bg-gray-600 text-white py-3 rounded-lg font-semibold"
                                        >
                                            ← Modificar
                                        </button>
                                        <button
                                            onClick={saveFlight}
                                            className="flex-1 bg-green-600 text-white py-3 rounded-lg font-semibold"
                                        >
                                            💾 Guardar Plan
                                        </button>
                                    </div>
                                </>
                            ) : null}
                        </div>
                    )}
                </div>
            </div>
        );
    }

    // ==================== APP PRINCIPAL ====================
    function App() {
        const [currentUser, setCurrentUser] = useState(null);
        const [currentView, setCurrentView] = useState('dashboard');

        useEffect(() => {
            // Registrar Service Worker
            if ('serviceWorker' in navigator) {
                navigator.serviceWorker.register('/service-worker.js')
                    .then(reg => console.log('Service Worker registrado'))
                    .catch(err => console.log('Error registrando SW:', err));
            }
        }, []);

        if (!currentUser) {
            return <LoginScreen onLogin={setCurrentUser} />;
        }

        if (currentView === 'dashboard') {
            return <Dashboard user={currentUser} onNewFlight={() => setCurrentView('planner')} />;
        }

        return <FlightPlanner user={currentUser} onBackToDashboard={() => setCurrentView('dashboard')} />;
    }

    ReactDOM.render(<App />, document.getElementById('root'));
</script>
```

</body>
</html>
{
“name”: “SEMAR Flight Planner”,
“short_name”: “SEMAR FP”,
“description”: “Sistema de Planificación de Vuelo para la Secretaría de Marina - Armada de México”,
“start_url”: “/”,
“display”: “standalone”,
“background_color”: “#0f172a”,
“theme_color”: “#3b82f6”,
“orientation”: “any”,
“icons”: [
{
“src”: “data:image/svg+xml,%3Csvg xmlns=‘http://www.w3.org/2000/svg’ viewBox=‘0 0 100 100’%3E%3Crect fill=’%233b82f6’ width=‘100’ height=‘100’/%3E%3Ctext x=‘50’ y=‘65’ font-size=‘60’ text-anchor=‘middle’ fill=‘white’%3E⚓%3C/text%3E%3C/svg%3E”,
“sizes”: “192x192”,
“type”: “image/svg+xml”,
“purpose”: “any maskable”
},
{
“src”: “data:image/svg+xml,%3Csvg xmlns=‘http://www.w3.org/2000/svg’ viewBox=‘0 0 100 100’%3E%3Crect fill=’%233b82f6’ width=‘100’ height=‘100’/%3E%3Ctext x=‘50’ y=‘65’ font-size=‘60’ text-anchor=‘middle’ fill=‘white’%3E⚓%3C/text%3E%3C/svg%3E”,
“sizes”: “512x512”,
“type”: “image/svg+xml”,
“purpose”: “any maskable”
}
],
“shortcuts”: [
{
“name”: “Nuevo Plan de Vuelo”,
“short_name”: “Nuevo”,
“description”: “Crear un nuevo plan de vuelo”,
“url”: “/?action=new”,
“icons”: [{ “src”: “data:image/svg+xml,%3Csvg xmlns=‘http://www.w3.org/2000/svg’ viewBox=‘0 0 100 100’%3E%3Ctext x=‘50’ y=‘65’ font-size=‘60’ text-anchor=‘middle’%3E✈️%3C/text%3E%3C/svg%3E”, “sizes”: “96x96” }]
},
{
“name”: “Dashboard”,
“short_name”: “Dashboard”,
“description”: “Ver estadísticas de vuelos”,
“url”: “/?view=dashboard”,
“icons”: [{ “src”: “data:image/svg+xml,%3Csvg xmlns=‘http://www.w3.org/2000/svg’ viewBox=‘0 0 100 100’%3E%3Ctext x=‘50’ y=‘65’ font-size=‘60’ text-anchor=‘middle’%3E📊%3C/text%3E%3C/svg%3E”, “sizes”: “96x96” }]
}
],
“categories”: [“navigation”, “utilities”, “productivity”],
“screenshots”: [
{
“src”: “data:image/svg+xml,%3Csvg xmlns=‘http://www.w3.org/2000/svg’ viewBox=‘0 0 1200 800’%3E%3Crect fill=’%230f172a’ width=‘1200’ height=‘800’/%3E%3Ctext x=‘600’ y=‘400’ font-size=‘80’ text-anchor=‘middle’ fill=’%233b82f6’%3ESEMAR Flight Planner%3C/text%3E%3C/svg%3E”,
“sizes”: “1200x800”,
“type”: “image/svg+xml”,
“form_factor”: “wide”
}
]
}
// service-worker.js - Service Worker para modo offline
const CACHE_NAME = ‘semar-flight-planner-v1.0’;
const URLS_TO_CACHE = [
‘/’,
‘/index.html’,
‘/manifest.json’
];

// Instalación del Service Worker
self.addEventListener(‘install’, (event) => {
event.waitUntil(
caches.open(CACHE_NAME)
.then((cache) => {
console.log(‘Cache abierto’);
return cache.addAll(URLS_TO_CACHE);
})
);
self.skipWaiting();
});

// Activación del Service Worker
self.addEventListener(‘activate’, (event) => {
event.waitUntil(
caches.keys().then((cacheNames) => {
return Promise.all(
cacheNames.map((cacheName) => {
if (cacheName !== CACHE_NAME) {
console.log(‘Eliminando cache antigua:’, cacheName);
return caches.delete(cacheName);
}
})
);
})
);
self.clients.claim();
});

// Estrategia: Network First, fallback to Cache
self.addEventListener(‘fetch’, (event) => {
// Solo cachear requests GET
if (event.request.method !== ‘GET’) return;

// Estrategia especial para APIs meteorológicas
if (event.request.url.includes(‘aviationweather.gov’) ||
event.request.url.includes(‘checkwxapi.com’)) {
event.respondWith(
fetch(event.request)
.then((response) => {
// Clonar respuesta para cachear
const responseToCache = response.clone();
caches.open(CACHE_NAME).then((cache) => {
cache.put(event.request, responseToCache);
});
return response;
})
.catch(() => {
// Si falla la red, usar cache
return caches.match(event.request);
})
);
} else {
// Para otros recursos: Cache First
event.respondWith(
caches.match(event.request)
.then((response) => {
if (response) {
return response;
}
return fetch(event.request).then((response) => {
// Cachear respuesta si es válida
if (!response || response.status !== 200 || response.type === ‘error’) {
return response;
}
const responseToCache = response.clone();
caches.open(CACHE_NAME).then((cache) => {
cache.put(event.request, responseToCache);
});
return response;
});
})
);
}
});

// Sincronización en background
self.addEventListener(‘sync’, (event) => {
if (event.tag === ‘sync-flight-data’) {
event.waitUntil(syncFlightData());
}
});

async function syncFlightData() {
// Aquí puedes sincronizar datos con el servidor cuando hay conexión
console.log(‘Sincronizando datos de vuelo…’);
}
