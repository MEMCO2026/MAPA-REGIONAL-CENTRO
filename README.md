<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mapa Sedes Colmédica & SURA - Bogotá y Chía</title>
    <meta name="description" content="Mapa interactivo de todas las sedes de Colmédica Medicina Prepagada y SURA EPS en Bogotá y Chía">
    
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    
    <!-- Leaflet CSS -->
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    
    <!-- Google Fonts -->
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600;700&display=swap" rel="stylesheet">
    
    <!-- Font Awesome -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">

    <style>
        body { font-family: 'Inter', sans-serif; }
        ::-webkit-scrollbar { width: 8px; }
        ::-webkit-scrollbar-track { background: #f1f1f1; }
        ::-webkit-scrollbar-thumb { background: #cbd5e1; border-radius: 4px; }
        
        .glass {
            background: rgba(255, 255, 255, 0.95);
            backdrop-filter: blur(10px);
            border: 1px solid rgba(255, 255, 255, 0.2);
        }

        @keyframes pulse-ring {
            0% { transform: scale(0.8); opacity: 0.5; }
            100% { transform: scale(2); opacity: 0; }
        }
        
        .pulse-ring::before {
            content: '';
            position: absolute;
            left: 50%;
            top: 50%;
            transform: translate(-50%, -50%);
            width: 100%;
            height: 100%;
            border-radius: 50%;
            border: 2px solid currentColor;
            animation: pulse-ring 2s cubic-bezier(0.215, 0.61, 0.355, 1) infinite;
        }

        .marker-pin {
            width: 40px;
            height: 40px;
            border-radius: 50% 50% 50% 0;
            position: absolute;
            transform: rotate(-45deg);
            left: 50%;
            top: 50%;
            margin: -20px 0 0 -20px;
            box-shadow: 0 4px 12px rgba(0,0,0,0.3);
            transition: all 0.3s ease;
            z-index: 100;
        }
        
        .marker-pin::after {
            content: '';
            width: 24px;
            height: 24px;
            margin: 8px 0 0 8px;
            background: white;
            position: absolute;
            border-radius: 50%;
            transform: rotate(45deg);
        }
        
        .marker-pin i {
            position: absolute;
            width: 100%;
            font-size: 16px;
            left: 0;
            top: 0;
            text-align: center;
            line-height: 40px;
            transform: rotate(45deg);
            z-index: 10;
            color: white;
        }

        .marker-pin.colmedica { background: #0ea5e9; }
        .marker-pin.sura { background: #f59e0b; }
        
        .marker-pin:hover {
            transform: rotate(-45deg) scale(1.2);
            box-shadow: 0 8px 25px rgba(0,0,0,0.4);
            z-index: 200;
        }

        .geo-label {
            background: rgba(255, 255, 255, 0.95);
            border: 2px solid;
            border-radius: 6px;
            padding: 4px 10px;
            font-size: 11px;
            font-weight: 700;
            white-space: nowrap;
            box-shadow: 0 2px 8px rgba(0,0,0,0.15);
            position: relative;
            pointer-events: none;
            z-index: 50;
            transition: all 0.3s ease;
            max-width: 200px;
            text-align: center;
            line-height: 1.2;
            display: block;
        }

        .geo-label.colmedica {
            border-color: #0ea5e9;
            color: #0369a1;
        }

        .geo-label.sura {
            border-color: #f59e0b;
            color: #b45309;
        }

        #map { height: 100vh; width: 100%; z-index: 1; }
        
        .sidebar { transition: transform 0.3s ease-in-out; }
        .sidebar.closed { transform: translateX(-100%); }
        
        .stat-card { transition: all 0.3s ease; }
        .stat-card:hover { transform: translateY(-5px); box-shadow: 0 10px 30px rgba(0,0,0,0.1); }
        
        .filter-btn { transition: all 0.3s ease; }
        .filter-btn.active { transform: scale(1.05); box-shadow: 0 4px 15px rgba(0,0,0,0.2); }

        .loader {
            border-top-color: #3498db;
            animation: spinner 1.5s linear infinite;
        }
        @keyframes spinner {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }

        @media (max-width: 768px) {
            .geo-label { font-size: 9px; padding: 2px 6px; max-width: 150px; }
        }

        .leaflet-popup-content-wrapper {
            border-radius: 12px;
            box-shadow: 0 10px 30px rgba(0,0,0,0.2);
        }
        .leaflet-popup-content { margin: 0; padding: 0; }
        .leaflet-container a.leaflet-popup-close-button {
            color: white;
            padding: 8px;
            font-size: 20px;
        }
    </style>
</head>
<body class="bg-gray-50 overflow-hidden">

    <!-- Loading Screen -->
    <div id="loader" class="fixed inset-0 bg-white z-50 flex items-center justify-center transition-opacity duration-500">
        <div class="text-center">
            <div class="loader ease-linear rounded-full border-4 border-t-4 border-gray-200 h-12 w-12 mb-4 mx-auto"></div>
            <h2 class="text-xl font-semibold text-gray-700">Cargando Mapa...</h2>
            <p class="text-sm text-gray-500 mt-2">Preparando sedes de Colmédica y SURA</p>
        </div>
    </div>

    <div class="relative h-screen w-full">
        <div id="map"></div>

        <!-- Header -->
        <div class="absolute top-4 left-4 right-4 md:left-4 md:right-auto z-[1000] glass rounded-2xl p-4 shadow-2xl max-w-md animate-fade-in">
            <div class="flex items-center justify-between mb-3">
                <div class="flex items-center space-x-3">
                    <div class="w-10 h-10 bg-gradient-to-br from-blue-500 to-blue-600 rounded-xl flex items-center justify-center text-white shadow-lg">
                        <i class="fas fa-map-marked-alt text-lg"></i>
                    </div>
                    <div>
                        <h1 class="text-lg font-bold text-gray-800 leading-tight">Mapa de Sedes</h1>
                        <p class="text-xs text-gray-500 font-medium">Bogotá & Chía</p>
                    </div>
                </div>
                <button onclick="toggleSidebar()" class="md:hidden p-2 hover:bg-gray-100 rounded-lg transition-colors">
                    <i class="fas fa-bars text-gray-600"></i>
                </button>
            </div>

            <!-- Stats -->
            <div class="grid grid-cols-3 gap-2 mb-4">
                <div class="stat-card bg-blue-50 rounded-xl p-2 text-center cursor-pointer" onclick="filterMarkers('all')">
                    <div class="text-lg font-bold text-blue-600" id="total-count">0</div>
                    <div class="text-[10px] text-blue-600 font-semibold uppercase tracking-wide">Total</div>
                </div>
                <div class="stat-card bg-sky-50 rounded-xl p-2 text-center cursor-pointer" onclick="filterMarkers('colmedica')">
                    <div class="text-lg font-bold text-sky-600" id="colmedica-count">0</div>
                    <div class="text-[10px] text-sky-600 font-semibold uppercase tracking-wide">Colmédica</div>
                </div>
                <div class="stat-card bg-amber-50 rounded-xl p-2 text-center cursor-pointer" onclick="filterMarkers('sura')">
                    <div class="text-lg font-bold text-amber-600" id="sura-count">0</div>
                    <div class="text-[10px] text-amber-600 font-semibold uppercase tracking-wide">SURA</div>
                </div>
            </div>

            <!-- Filters -->
            <div class="flex space-x-2 mb-3">
                <button onclick="filterMarkers('all')" id="btn-all" class="filter-btn active flex-1 bg-gray-800 text-white py-2 px-4 rounded-xl text-xs font-semibold hover:bg-gray-700 transition-all">
                    <i class="fas fa-layer-group mr-1"></i> Todas
                </button>
                <button onclick="filterMarkers('colmedica')" id="btn-colmedica" class="filter-btn flex-1 bg-sky-500 text-white py-2 px-4 rounded-xl text-xs font-semibold hover:bg-sky-600 transition-all">
                    <i class="fas fa-heartbeat mr-1"></i> Colmédica
                </button>
                <button onclick="filterMarkers('sura')" id="btn-sura" class="filter-btn flex-1 bg-amber-500 text-white py-2 px-4 rounded-xl text-xs font-semibold hover:bg-amber-600 transition-all">
                    <i class="fas fa-shield-alt mr-1"></i> SURA
                </button>
            </div>

            <!-- Toggle Labels -->
            <button onclick="toggleLabels()" id="btn-labels" class="w-full flex items-center justify-center space-x-2 bg-white border border-gray-300 text-gray-700 py-2 rounded-xl text-xs font-semibold hover:bg-gray-50 transition-all">
                <i class="fas fa-tag text-green-500"></i>
                <span>Ocultar nombres de sedes</span>
            </button>
        </div>

        <!-- Sidebar -->
        <div id="sidebar" class="sidebar absolute top-4 left-4 bottom-4 w-80 glass rounded-2xl shadow-2xl z-[1000] overflow-hidden flex flex-col transform -translate-x-full md:translate-x-0 transition-transform duration-300 mt-24 md:mt-0 md:top-32 md:left-4 md:bottom-4 md:h-auto">
            <div class="p-4 border-b border-gray-200 bg-white/50">
                <div class="flex items-center justify-between">
                    <h2 class="font-bold text-gray-800 flex items-center">
                        <i class="fas fa-list-ul mr-2 text-blue-500"></i>
                        Lista de Sedes
                    </h2>
                    <button onclick="toggleSidebar()" class="md:hidden p-1 hover:bg-gray-200 rounded-lg">
                        <i class="fas fa-times text-gray-600"></i>
                    </button>
                </div>
                <div class="mt-2 relative">
                    <input type="text" id="search-input" placeholder="Buscar sede..." 
                           class="w-full pl-9 pr-4 py-2 bg-white border border-gray-200 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent transition-all"
                           onkeyup="searchLocations()">
                    <i class="fas fa-search absolute left-3 top-2.5 text-gray-400 text-xs"></i>
                </div>
            </div>

            <div id="locations-list" class="flex-1 overflow-y-auto p-3 space-y-2"></div>

            <div class="p-3 border-t border-gray-200 bg-white/50 text-center">
                <p class="text-[10px] text-gray-500">Actualizado: Marzo 2026</p>
            </div>
        </div>

        <!-- Toggle Sidebar Button -->
        <button onclick="toggleSidebar()" class="hidden md:flex absolute top-32 left-4 z-[1000] w-10 h-10 bg-white rounded-full shadow-lg items-center justify-center hover:bg-gray-50 transition-all transform hover:scale-110">
            <i class="fas fa-chevron-right text-gray-600" id="sidebar-toggle-icon"></i>
        </button>

        <!-- Legend -->
        <div class="absolute bottom-4 right-4 z-[1000] glass rounded-xl p-3 shadow-lg animate-fade-in">
            <h3 class="text-xs font-bold text-gray-700 mb-2 uppercase tracking-wide">Leyenda</h3>
            <div class="space-y-2">
                <div class="flex items-center space-x-2">
                    <div class="w-3 h-3 rounded-full bg-sky-500 shadow-sm"></div>
                    <span class="text-xs text-gray-600 font-medium">Colmédica Medicina Prepagada</span>
                </div>
                <div class="flex items-center space-x-2">
                    <div class="w-3 h-3 rounded-full bg-amber-500 shadow-sm"></div>
                    <span class="text-xs text-gray-600 font-medium">SURA EPS / Salud</span>
                </div>
            </div>
        </div>

        <!-- Detail Modal -->
        <div id="detail-modal" class="fixed inset-0 bg-black/50 z-[2000] hidden items-center justify-center p-4 backdrop-blur-sm transition-opacity duration-300">
            <div class="bg-white rounded-2xl shadow-2xl max-w-sm w-full transform scale-95 opacity-0 transition-all duration-300" id="modal-content">
                <div class="relative h-32 bg-gradient-to-br from-blue-500 to-blue-600 rounded-t-2xl overflow-hidden">
                    <div class="absolute inset-0 bg-black/10"></div>
                    <button onclick="closeModal()" class="absolute top-3 right-3 w-8 h-8 bg-white/20 hover:bg-white/30 rounded-full flex items-center justify-center text-white transition-colors">
                        <i class="fas fa-times"></i>
                    </button>
                    <div class="absolute bottom-4 left-4 text-white">
                        <span id="modal-type" class="text-[10px] font-bold uppercase tracking-wider bg-white/20 px-2 py-1 rounded-full backdrop-blur-sm">Tipo</span>
                        <h3 id="modal-title" class="text-xl font-bold mt-1">Nombre Sede</h3>
                    </div>
                </div>
                <div class="p-5">
                    <div class="space-y-3">
                        <div class="flex items-start space-x-3">
                            <div class="w-8 h-8 rounded-lg bg-gray-100 flex items-center justify-center flex-shrink-0 mt-0.5">
                                <i class="fas fa-map-marker-alt text-gray-500 text-sm"></i>
                            </div>
                            <div>
                                <p class="text-xs text-gray-500 font-semibold uppercase">Dirección</p>
                                <p id="modal-address" class="text-sm text-gray-800 font-medium leading-tight">Dirección completa</p>
                            </div>
                        </div>
                        
                        <div class="flex items-start space-x-3">
                            <div class="w-8 h-8 rounded-lg bg-gray-100 flex items-center justify-center flex-shrink-0 mt-0.5">
                                <i class="fas fa-phone text-gray-500 text-sm"></i>
                            </div>
                            <div>
                                <p class="text-xs text-gray-500 font-semibold uppercase">Teléfono</p>
                                <p id="modal-phone" class="text-sm text-gray-800 font-medium">Teléfono</p>
                            </div>
                        </div>

                        <div class="flex items-start space-x-3">
                            <div class="w-8 h-8 rounded-lg bg-gray-100 flex items-center justify-center flex-shrink-0 mt-0.5">
                                <i class="fas fa-clock text-gray-500 text-sm"></i>
                            </div>
                            <div>
                                <p class="text-xs text-gray-500 font-semibold uppercase">Horario</p>
                                <p id="modal-hours" class="text-sm text-gray-800 font-medium">Horario de atención</p>
                            </div>
                        </div>
                    </div>

                    <div class="mt-5 flex space-x-2">
                        <button onclick="openInMaps()" class="flex-1 bg-gray-900 text-white py-2.5 rounded-xl text-xs font-semibold hover:bg-gray-800 transition-colors flex items-center justify-center">
                            <i class="fas fa-external-link-alt mr-2"></i>
                            Abrir en Maps
                        </button>
                        <button onclick="closeModal()" class="px-4 py-2.5 border border-gray-200 rounded-xl text-xs font-semibold text-gray-600 hover:bg-gray-50 transition-colors">
                            Cerrar
                        </button>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    
    <script>
        const locations = [
            { id: 1, name: "Oficina Principal Calle 93", type: "colmedica", address: "Cl. 93 #19-25 Piso 1, Chapinero, Bogotá", phone: "(601) 7464646", hours: "Lun-Vie: 7:30am - 5:00pm, Sáb: 8:00am - 1:00pm", coords: [4.6769, -74.0489], category: "Administrativa" },
            { id: 2, name: "Centro Médico Torre Santa Bárbara", type: "colmedica", address: "Autopista Norte #122-96, Santa Bárbara, Bogotá", phone: "(601) 7464646", hours: "Dom-Dom: 7:00am - 7:00pm", coords: [4.7029, -74.0349], category: "Centro Médico" },
            { id: 3, name: "Centro Médico Colina Campestre", type: "colmedica", address: "CC Sendero de la Colina, Cl 151 #54-15 Piso 2, Bogotá", phone: "(601) 7464646", hours: "Lun-Dom: 7:00am - 7:00pm", coords: [4.7328, -74.0519], category: "Centro Médico" },
            { id: 4, name: "Centro Médico Chapinero", type: "colmedica", address: "Cr 7 #52-53, Chapinero, Bogotá", phone: "(601) 7464646", hours: "Lun-Sáb: 7:00am - 7:00pm", coords: [4.6392, -74.0625], category: "Centro Médico" },
            { id: 5, name: "Centro Médico Cedritos", type: "colmedica", address: "Cl 140 #11-45 Piso 9, Cedritos, Bogotá", phone: "(601) 7464646", hours: "Lun-Sáb: 7:00am - 6:00pm", coords: [4.7189, -74.0378], category: "Centro Médico" },
            { id: 6, name: "Centro Médico Suba", type: "colmedica", address: "Av Cl 145 #103B-69, Centro Empresarial Alpaso Plaza, Bogotá", phone: "(601) 7464646", hours: "Lun-Sáb: 7:00am - 7:00pm", coords: [4.7456, -74.0945], category: "Centro Médico" },
            { id: 7, name: "Centro Médico Bulevar Niza", type: "colmedica", address: "CC Bulevar Niza, Av Cr 58 #127-59 Piso 2, Bogotá", phone: "(601) 7464646", hours: "Lun-Dom: 8:00am - 8:00pm", coords: [4.7123, -74.0723], category: "Centro Médico" },
            { id: 8, name: "Centro Médico Plaza Central", type: "colmedica", address: "Cr 65 #11-50 Local 4-27A, CC Plaza Central Piso 4, Bogotá", phone: "(601) 7464646", hours: "Lun-Dom: 7:00am - 7:00pm", coords: [4.6345, -74.1187], category: "Centro Médico" },
            { id: 9, name: "Centro Médico Salitre Capital", type: "colmedica", address: "Av Cl 26 #69C-03, Edificio Capital Center II, Bogotá", phone: "(601) 7464646", hours: "Lun-Vie: 7:00am - 7:00pm", coords: [4.6512, -74.1023], category: "Centro Médico" },
            { id: 10, name: "Centro Médico Country Park", type: "colmedica", address: "Cr 18 #84-11 Pisos 2 y 3, Bogotá", phone: "(601) 7464646", hours: "Lun-Sáb: 7:00am - 6:00pm", coords: [4.6712, -74.0567], category: "Centro Médico" },
            { id: 11, name: "Centro Médico Calle 185", type: "colmedica", address: "Cl 185 #45-03, CC Santafé Piso 2, Bogotá", phone: "(601) 7464646", hours: "Lun-Dom: 10:00am - 9:00pm", coords: [4.7623, -74.0456], category: "Centro Médico" },
            { id: 12, name: "Centro Médico Metrópolis", type: "colmedica", address: "Av Cr 68 #75A-50, CC Metrópolis Local 262A, Bogotá", phone: "(601) 7464646", hours: "Lun-Dom: 10:00am - 9:00pm", coords: [4.6789, -74.0823], category: "Centro Médico" },
            { id: 13, name: "Centro Médico Belaire", type: "colmedica", address: "Cl 153 #6-65 Local 201, CC Belaire Plaza, Bogotá", phone: "(601) 7464646", hours: "Lun-Sáb: 8:00am - 6:00pm", coords: [4.7289, -74.0234], category: "Centro Médico" },
            { id: 14, name: "Centro Médico Unicentro de Occidente", type: "colmedica", address: "Cr 111C #86-05, CC Unicentro de Occidente Piso 2, Bogotá", phone: "(601) 7464646", hours: "Lun-Dom: 10:00am - 9:00pm", coords: [4.7123, -74.1234], category: "Centro Médico" },
            { id: 15, name: "Centro de Diagnóstico Bella Suiza", type: "colmedica", address: "Cl 127A #7-19, Bogotá", phone: "(601) 7464646", hours: "Lun-Sáb: 7:00am - 5:00pm", coords: [4.7034, -74.0345], category: "Diagnóstico" },
            { id: 16, name: "Centro Médico Colmédica Chía", type: "colmedica", address: "Chía, Cundinamarca (Centro Comercial)", phone: "(601) 7464646", hours: "Lun-Sáb: 8:00am - 6:00pm", coords: [4.8589, -74.0512], category: "Centro Médico" },
            { id: 17, name: "Salud SURA Usaquén", type: "sura", address: "Cl 119A #7-74, Usaquén, Bogotá", phone: "(601) 482 4366", hours: "Lun-Vie: 7:00am - 7:00pm, Sáb: 7:00am - 1:00pm", coords: [4.6945, -74.0345], category: "IPS" },
            { id: 18, name: "Sede Calle 100", type: "sura", address: "Cl 100 #19A-35, Bogotá", phone: "(601) 487 3888", hours: "Lun-Vie: 7:00am - 6:00pm", coords: [4.6845, -74.0512], category: "Administrativa" },
            { id: 19, name: "Ayudas Diagnósticas SURA - Plaza Central", type: "sura", address: "Cr 65 #11-50 Local 2-48, CC Plaza Central, Bogotá", phone: "(601) 489 7941", hours: "Lun-Sáb: 7:00am - 6:00pm", coords: [4.6340, -74.1180], category: "Diagnóstico" },
            { id: 20, name: "SF SURA Chapinero", type: "sura", address: "Cr 7 #56-29, Chapinero, Bogotá", phone: "(601) 489 7941", hours: "Lun-Vie: 7:00am - 9:00pm, Sáb: 7:00am - 6:00pm", coords: [4.6423, -74.0612], category: "Punto de Servicio" },
            { id: 21, name: "SF SURA Niza", type: "sura", address: "Cl 127A #70D-60 CC Niza LC 53, Bogotá", phone: "(601) 489 7941", hours: "Lun-Vie: 7:00am - 9:00pm, Sáb: 7:00am - 4:00pm", coords: [4.7089, -74.0723], category: "Punto de Servicio" },
            { id: 22, name: "SF SURA Sur", type: "sura", address: "Cr 13 #2-56 Sur, Bogotá", phone: "(601) 489 7941", hours: "Lun-Vie: 7:00am - 9:00pm, Sáb: 7:00am - 4:00pm", coords: [4.5856, -74.0890], category: "Punto de Servicio" },
            { id: 23, name: "SF SURA Américas", type: "sura", address: "Cl 6 #71B-30, Bogotá", phone: "(601) 489 7941", hours: "Lun-Vie: 7:00am - 9:00pm, Sáb: 7:00am - 4:00pm", coords: [4.6234, -74.1234], category: "Punto de Servicio" },
            { id: 24, name: "SF SURA Kennedy", type: "sura", address: "Cl 35 Sur #78K-17, Kennedy, Bogotá", phone: "(601) 489 7941", hours: "Lun-Vie: 7:00am - 7:00pm, Sáb: 7:00am - 4:00pm", coords: [4.6234, -74.1567], category: "Punto de Servicio" },
            { id: 25, name: "SF SURA Centro Internacional", type: "sura", address: "Cl 31 #13A-52 Torre I Sótano 2, Edif. Panorama, Bogotá", phone: "(601) 489 7941", hours: "Lun-Vie: 7:00am - 7:00pm, Sáb: 7:00am - 5:00pm", coords: [4.6156, -74.0723], category: "Punto de Servicio" },
            { id: 26, name: "SF SURA Suba Acuarela", type: "sura", address: "Cl 145 #92-30 Local 006, Suba, Bogotá", phone: "(601) 489 7941", hours: "Lun-Sáb: 6:00am - 8:00pm", coords: [4.7456, -74.0890], category: "Punto de Servicio" },
            { id: 27, name: "SF SURA Av. Boyacá", type: "sura", address: "Cr 73Bis #64A-80, Bogotá", phone: "(601) 489 7941", hours: "Lun-Vie: 7:00am - 8:00pm, Sáb: 7:00am - 2:00pm", coords: [4.6789, -74.0890], category: "Punto de Servicio" },
            { id: 28, name: "EPS SURA Chía", type: "sura", address: "Cr 153 #132, Chía, Cundinamarca", phone: "(601) 8844432", hours: "Lun-Vie: 8:00am - 5:00pm", coords: [4.8634, -74.0456], category: "Oficina EPS" },
            { id: 29, name: "SURA Centro Chía", type: "sura", address: "Centro Comercial Centro Chía Local 2006 Piso 2, Chía", phone: "8620559 - 8620083", hours: "Lun-Vie: 8:00am - 6:00pm, Sáb: 9:00am - 1:00pm", coords: [4.8580, -74.0400], category: "Centro de Servicios" }
        ];

        let map, markers = [], labelMarkers = [], currentFilter = 'all', currentLocation = null, labelsVisible = true;

        function initMap() {
            map = L.map('map', { zoomControl: false, attributionControl: false }).setView([4.72, -74.07], 11);
            L.control.zoom({ position: 'bottomright' }).addTo(map);
            L.tileLayer('https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png', {
                attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors &copy; <a href="https://carto.com/attributions">CARTO</a>',
                subdomains: 'abcd', maxZoom: 20
            }).addTo(map);

            renderMarkers();
            updateCounts();
            renderSidebarList();

            setTimeout(() => {
                document.getElementById('loader').style.opacity = '0';
                setTimeout(() => { document.getElementById('loader').style.display = 'none'; }, 500);
            }, 1000);
        }

        function createCustomIcon(type) {
            const color = type === 'colmedica' ? 'colmedica' : 'sura';
            const iconClass = type === 'colmedica' ? 'fa-heartbeat' : 'fa-shield-alt';
            return L.divIcon({
                className: 'custom-div-icon',
                html: `<div class="marker-pin ${color} pulse-ring"><i class="fas ${iconClass}"></i></div>`,
                iconSize: [40, 40], iconAnchor: [20, 40], popupAnchor: [0, -40]
            });
        }

        function createLabelIcon(loc) {
            return L.divIcon({
                className: 'custom-div-icon',
                html: `<div class="geo-label ${loc.type}">${loc.name}</div>`,
                iconSize: [200, 30], iconAnchor: [100, 15], popupAnchor: [0, -10]
            });
        }

        function renderMarkers() {
            markers.forEach(m => map.removeLayer(m));
            labelMarkers.forEach(l => map.removeLayer(l));
            markers = []; labelMarkers = [];

            locations.forEach((loc, index) => {
                const marker = L.marker(loc.coords, { icon: createCustomIcon(loc.type), zIndexOffset: 100 }).addTo(map);
                const popupContent = `
                    <div class="p-3 min-w-[250px]">
                        <h3 class="font-bold text-gray-800 text-sm mb-1">${loc.name}</h3>
                        <p class="text-xs text-gray-500 mb-2">${loc.address}</p>
                        <div class="flex space-x-2">
                            <button onclick="showDetail(${loc.id})" class="flex-1 bg-blue-500 text-white text-xs py-1.5 rounded-lg hover:bg-blue-600 transition-colors">Ver detalles</button>
                            <button onclick="zoomToLocation(${loc.id})" class="px-3 bg-gray-100 text-gray-700 text-xs py-1.5 rounded-lg hover:bg-gray-200 transition-colors"><i class="fas fa-search-plus"></i></button>
                        </div>
                    </div>`;
                
                marker.bindPopup(popupContent, { closeButton: true, offset: [0, -40], className: 'custom-popup' });
                marker.locationId = loc.id; marker.locationType = loc.type;
                markers.push(marker);
                marker.on('click', () => showDetail(loc.id));

                const labelMarker = L.marker(loc.coords, { icon: createLabelIcon(loc), zIndexOffset: 50, interactive: false });
                labelMarker.addTo(map); labelMarker.locationType = loc.type;
                labelMarkers.push(labelMarker);
            });

            map.on('zoomend', adjustLabelVisibility);
            adjustLabelVisibility();
        }

        function adjustLabelVisibility() {
            const zoom = map.getZoom();
            labelMarkers.forEach((labelMarker, index) => {
                const element = labelMarker.getElement();
                if (!element) return;
                if (!labelsVisible) { element.style.display = 'none'; return; }
                
                if (zoom < 13) {
                    element.style.opacity = '0'; element.style.pointerEvents = 'none';
                } else {
                    element.style.opacity = '1'; element.style.display = 'block';
                    const offset = index % 2 === 0 ? -35 : 35;
                    const labelDiv = element.querySelector('.geo-label');
                    if (labelDiv) labelDiv.style.transform = `translateY(${offset}px)`;
                }
            });
        }

        function zoomToLocation(id) {
            const loc = locations.find(l => l.id === id);
            if (loc) map.flyTo(loc.coords, 18, { duration: 1.5 });
        }

        function toggleLabels() {
            labelsVisible = !labelsVisible;
            const btn = document.getElementById('btn-labels');
            const icon = btn.querySelector('i'), text = btn.querySelector('span');
            adjustLabelVisibility();
            
            if (labelsVisible) {
                icon.className = 'fas fa-tag text-green-500'; text.textContent = 'Ocultar nombres de sedes';
                btn.classList.remove('bg-red-50', 'border-red-200', 'text-red-700');
                btn.classList.add('bg-white', 'border-gray-300', 'text-gray-700');
            } else {
                icon.className = 'fas fa-tag text-red-500'; text.textContent = 'Mostrar nombres de sedes';
                btn.classList.remove('bg-white', 'border-gray-300', 'text-gray-700');
                btn.classList.add('bg-red-50', 'border-red-200', 'text-red-700');
            }
        }

        function filterMarkers(type) {
            currentFilter = type;
            document.querySelectorAll('.filter-btn').forEach(btn => btn.classList.remove('active'));
            document.getElementById(`btn-${type}`).classList.add('active');

            markers.forEach(marker => {
                if (type === 'all' || marker.locationType === type) marker.addTo(map);
                else map.removeLayer(marker);
            });

            labelMarkers.forEach((labelMarker, index) => {
                const loc = locations[index];
                if (type === 'all' || loc.type === type) { if (labelsVisible) labelMarker.addTo(map); }
                else map.removeLayer(labelMarker);
            });
            renderSidebarList();
        }

        function updateCounts() {
            document.getElementById('total-count').textContent = locations.length;
            document.getElementById('colmedica-count').textContent = locations.filter(l => l.type === 'colmedica').length;
            document.getElementById('sura-count').textContent = locations.filter(l => l.type === 'sura').length;
        }

        function renderSidebarList() {
            const list = document.getElementById('locations-list');
            const searchTerm = document.getElementById('search-input').value.toLowerCase();
            list.innerHTML = '';

            const filtered = locations.filter(loc => {
                const matchesType = currentFilter === 'all' || loc.type === currentFilter;
                const matchesSearch = loc.name.toLowerCase().includes(searchTerm) || loc.address.toLowerCase().includes(searchTerm);
                return matchesType && matchesSearch;
            });

            filtered.forEach(loc => {
                const item = document.createElement('div');
                item.className = `p-3 rounded-xl cursor-pointer transition-all hover:shadow-md border border-transparent hover:border-gray-200 ${loc.type === 'colmedica' ? 'bg-sky-50/50 hover:bg-sky-50' : 'bg-amber-50/50 hover:bg-amber-50'}`;
                item.innerHTML = `
                    <div class="flex items-start space-x-3">
                        <div class="w-8 h-8 rounded-lg ${loc.type === 'colmedica' ? 'bg-sky-500' : 'bg-amber-500'} flex items-center justify-center flex-shrink-0 text-white">
                            <i class="fas ${loc.type === 'colmedica' ? 'fa-heartbeat' : 'fa-shield-alt'} text-xs"></i>
                        </div>
                        <div class="flex-1 min-w-0">
                            <h4 class="text-sm font-bold text-gray-800 truncate">${loc.name}</h4>
                            <p class="text-xs text-gray-500 mt-0.5 line-clamp-2">${loc.address}</p>
                            <span class="inline-block mt-1.5 text-[10px] px-2 py-0.5 rounded-full ${loc.type === 'colmedica' ? 'bg-sky-100 text-sky-700' : 'bg-amber-100 text-amber-700'} font-semibold">${loc.category}</span>
                        </div>
                    </div>`;
                item.onclick = () => { showDetail(loc.id); map.flyTo(loc.coords, 16, { duration: 1.5 }); if (window.innerWidth < 768) toggleSidebar(); };
                list.appendChild(item);
            });
        }

        function searchLocations() { renderSidebarList(); }

        function showDetail(id) {
            const loc = locations.find(l => l.id === id);
            if (!loc) return;
            currentLocation = loc;
            
            document.getElementById('modal-title').textContent = loc.name;
            document.getElementById('modal-type').textContent = loc.type === 'colmedica' ? 'Colmédica' : 'SURA EPS';
            document.getElementById('modal-address').textContent = loc.address;
            document.getElementById('modal-phone').textContent = loc.phone;
            document.getElementById('modal-hours').textContent = loc.hours;
            
            const header = document.querySelector('#modal-content > div:first-child');
            if (loc.type === 'colmedica') {
                header.className = 'relative h-32 bg-gradient-to-br from-sky-500 to-blue-600 rounded-t-2xl overflow-hidden';
                document.getElementById('modal-type').className = 'text-[10px] font-bold uppercase tracking-wider bg-white/20 px-2 py-1 rounded-full backdrop-blur-sm text-white';
            } else {
                header.className = 'relative h-32 bg-gradient-to-br from-amber-500 to-orange-600 rounded-t-2xl overflow-hidden';
                document.getElementById('modal-type').className = 'text-[10px] font-bold uppercase tracking-wider bg-white/20 px-2 py-1 rounded-full backdrop-blur-sm text-white';
            }
            
            const modal = document.getElementById('detail-modal'), content = document.getElementById('modal-content');
            modal.classList.remove('hidden'); modal.classList.add('flex');
            setTimeout(() => { content.classList.remove('scale-95', 'opacity-0'); content.classList.add('scale-100', 'opacity-100'); }, 10);
        }

        function closeModal() {
            const modal = document.getElementById('detail-modal'), content = document.getElementById('modal-content');
            content.classList.remove('scale-100', 'opacity-100'); content.classList.add('scale-95', 'opacity-0');
            setTimeout(() => { modal.classList.add('hidden'); modal.classList.remove('flex'); }, 300);
        }

        function openInMaps() {
            if (currentLocation) window.open(`https://www.google.com/maps/search/?api=1&query=${currentLocation.coords[0]},${currentLocation.coords[1]}`, '_blank');
        }

        function toggleSidebar() {
            const sidebar = document.getElementById('sidebar'), icon = document.getElementById('sidebar-toggle-icon');
            sidebar.classList.toggle('closed');
            if (sidebar.classList.contains('closed')) { icon.classList.remove('fa-chevron-left'); icon.classList.add('fa-chevron-right'); }
            else { icon.classList.remove('fa-chevron-right'); icon.classList.add('fa-chevron-left'); }
        }

        document.getElementById('detail-modal').addEventListener('click', function(e) { if (e.target === this) closeModal(); });
        window.onload = initMap;
    </script>
</body>
</html>
