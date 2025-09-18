<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Reportes de Fugas y Fallas - COMAPA</title>
  <link
    rel="stylesheet"
    href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
  />
  <style>
    body, html {
      margin: 0;
      padding: 0;
      height: 100%;
      font-family: Arial, sans-serif;
      background: #f4f4f9;
    }
    #map {
      height: 100%;
      width: 100%;
    }
    .form-popup, .message-modal {
      position: fixed;
      bottom: 20px;
      left: 50%;
      transform: translateX(-50%);
      z-index: 1000;
      background: white;
      padding: 16px;
      border-radius: 12px;
      box-shadow: 0 6px 16px rgba(0, 0, 0, 0.15);
      width: 90%;
      max-width: 400px;
      display: none;
      text-align: center;
    }
    .form-popup input, .form-popup select, .form-popup textarea {
      width: 100%;
      padding: 10px;
      margin-bottom: 12px;
      border-radius: 6px;
      border: 1px solid #ccc;
      font-size: 1em;
    }
    .form-popup button, .message-modal button {
      width: 100%;
      padding: 10px;
      margin-top: 4px;
      border: none;
      border-radius: 6px;
      font-size: 1em;
      cursor: pointer;
    }
    #saveBtn {
      background: #28a745;
      color: white;
    }
    #saveBtn:hover {
      background: #218838;
    }
    #cancelBtn, .message-modal button {
      background: #dee2e6;
      color: #333;
    }
    #cancelBtn:hover, .message-modal button:hover {
      background: #c6cbd1;
    }
    .popup-content button {
        background: #007bff;
        color: white;
        border: none;
        padding: 5px 10px;
        border-radius: 5px;
        cursor: pointer;
        font-size: 0.9em;
    }
    .popup-content button:hover {
        background: #0056b3;
    }
    .popup-content button:nth-of-type(2) {
        background: #dc3545;
    }
    .popup-content button:nth-of-type(2):hover {
        background: #c82333;
    }
    .leaflet-container .leaflet-marker-icon {
        background-size: contain;
    }
    .leaflet-container .leak-icon {
        background-image: url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="32" height="32" viewBox="0 0 24 24" fill="%23007bff"><path d="M12 21.35l-1.45-1.32C5.4 15.36 2 12.28 2 8.5 2 5.42 4.42 3 7.5 3c1.74 0 3.41.81 4.5 2.09C13.09 3.81 14.76 3 16.5 3 19.58 3 22 5.42 22 8.5c0 3.78-3.4 6.86-8.55 11.54L12 21.35z"/></svg>');
    }
    .leaflet-container .failure-icon {
        background-image: url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="32" height="32" viewBox="0 0 24 24" fill="%23ff8c00"><path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 15h-2v-2h2v2zm0-4h-2V7h2v6z"/></svg>');
    }
    .loading-spinner {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      border: 4px solid rgba(0, 0, 0, 0.1);
      border-top: 4px solid #3498db;
      border-radius: 50%;
      width: 40px;
      height: 40px;
      animation: spin 1s linear infinite;
      z-index: 1001;
      display: none;
    }
    @keyframes spin {
      0% { transform: rotate(0deg); }
      100% { transform: rotate(360deg); }
    }
    .photo-preview {
        max-width: 100%;
        height: auto;
        margin-top: 10px;
        border-radius: 8px;
    }
  </style>
</head>
<body>
  <div id="map"></div>
  <div class="loading-spinner" id="spinner"></div>

  <div class="form-popup" id="reportForm">
    <p><b>Reportar una incidencia</b></p>
    <select id="reportType">
      <option value="fuga">Fuga de Agua</option>
      <option value="falla">Falla en Infraestructura</option>
    </select>
    <textarea id="description" rows="3" placeholder="Describe brevemente la incidencia..."></textarea>
    <label for="photoUpload" style="display: block; margin-bottom: 12px; cursor: pointer; background: #6c757d; color: white; padding: 10px; border-radius: 6px;">
        📸 Subir Foto
    </label>
    <input type="file" id="photoUpload" accept="image/*" style="display: none;">
    <img id="imagePreview" class="photo-preview" style="display: none;">
    <button id="saveBtn">💾 Enviar Reporte</button>
    <button id="cancelBtn">❌ Cancelar</button>
  </div>

  <div class="message-modal" id="messageModal">
    <p id="messageText"></p>
    <button id="okBtn">OK</button>
  </div>

  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
    import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
    import { getFirestore, doc, getDoc, addDoc, setDoc, updateDoc, deleteDoc, onSnapshot, collection, query, where, getDocs } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

    // Variables globales proporcionadas por el entorno de Canvas
    const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
    const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
    const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

    let app, auth, db, userId;
    let map = L.map("map").setView([23.767, -98.216], 13);
    let currentLatLng = null;
    let editingReportId = null;
    let reportMarkers = {};
    const reportForm = document.getElementById("reportForm");
    const spinner = document.getElementById("spinner");
    const messageModal = document.getElementById("messageModal");
    const messageText = document.getElementById("messageText");
    const okBtn = document.getElementById("okBtn");
    const photoUploadInput = document.getElementById("photoUpload");
    const imagePreview = document.getElementById("imagePreview");
    let uploadedFile = null;
    
    // Iconos personalizados para los reportes
    const LeakIcon = L.Icon.extend({
        options: {
            iconUrl: 'data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="32" height="32" viewBox="0 0 24 24" fill="%23007bff"><path d="M12 21.35l-1.45-1.32C5.4 15.36 2 12.28 2 8.5 2 5.42 4.42 3 7.5 3c1.74 0 3.41.81 4.5 2.09C13.09 3.81 14.76 3 16.5 3 19.58 3 22 5.42 22 8.5c0 3.78-3.4 6.86-8.55 11.54L12 21.35z"/></svg>',
            iconSize: [32, 32],
            iconAnchor: [16, 32],
            popupAnchor: [0, -25]
        }
    });

    const FailureIcon = L.Icon.extend({
        options: {
            iconUrl: 'data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="32" height="32" viewBox="0 0 24 24" fill="%23ff8c00"><path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 15h-2v-2h2v2zm0-4h-2V7h2v6z"/></svg>',
            iconSize: [32, 32],
            iconAnchor: [16, 32],
            popupAnchor: [0, -25]
        }
    });

    const leakIcon = new LeakIcon();
    const failureIcon = new FailureIcon();

    // Inicializa el mapa y la capa base
    L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
      attribution: "© OpenStreetMap contributors",
    }).addTo(map);

    // Muestra un modal de mensaje simple
    function showMessage(message) {
      messageText.textContent = message;
      messageModal.style.display = 'block';
      okBtn.onclick = () => messageModal.style.display = 'none';
    }

    // Muestra un spinner de carga
    function showSpinner() {
      spinner.style.display = 'block';
    }

    // Oculta el spinner de carga
    function hideSpinner() {
      spinner.style.display = 'none';
    }

    // Inicializa Firebase y autentica al usuario
    async function initFirebase() {
      try {
        app = initializeApp(firebaseConfig);
        auth = getAuth(app);
        db = getFirestore(app);

        // Intenta autenticar con el token personalizado, si no, usa autenticación anónima
        if (initialAuthToken) {
          await signInWithCustomToken(auth, initialAuthToken);
        } else {
          await signInAnonymously(auth);
        }

        onAuthStateChanged(auth, (user) => {
          if (user) {
            userId = user.uid;
            console.log("Usuario autenticado. ID:", userId);
            fetchReports();
          } else {
            console.error("No se pudo autenticar al usuario.");
          }
        });
      } catch (error) {
        console.error("Error al inicializar Firebase:", error);
        showMessage("Error al conectar con la base de datos.");
      }
    }

    // Escucha los reportes en tiempo real desde Firestore
    function fetchReports() {
      showSpinner();
      const reportsCollection = collection(db, `artifacts/${appId}/public/data/reports`);
      onSnapshot(reportsCollection, (snapshot) => {
        snapshot.docChanges().forEach(change => {
          const report = { id: change.doc.id, ...change.doc.data() };
          if (change.type === "added") {
            createMarker(report);
          }
          if (change.type === "modified") {
            updateMarker(report);
          }
          if (change.type === "removed") {
            deleteMarker(report.id);
          }
        });
        hideSpinner();
      }, (error) => {
        console.error("Error al obtener los reportes:", error);
        showMessage("No se pudieron cargar los reportes. Revisa la consola para más detalles.");
        hideSpinner();
      });
    }

    // Crea un marcador en el mapa
    function createMarker(report) {
      const icon = report.type === 'fuga' ? leakIcon : failureIcon;
      const marker = L.marker([report.location.lat, report.location.lng], { icon: icon }).addTo(map);
      marker.bindPopup(createPopupContent(report));
      marker.on("popupopen", () => {
        document.getElementById(`edit-${report.id}`).addEventListener("click", () => editReport(report.id));
        document.getElementById(`delete-${report.id}`).addEventListener("click", () => deleteReport(report.id));
      });
      reportMarkers[report.id] = marker;
    }

    // Actualiza un marcador existente en el mapa
    function updateMarker(report) {
      const marker = reportMarkers[report.id];
      if (marker) {
        const icon = report.type === 'fuga' ? leakIcon : failureIcon;
        marker.setLatLng([report.location.lat, report.location.lng]);
        marker.setIcon(icon);
        marker.setPopupContent(createPopupContent(report));
      }
    }

    // Elimina un marcador del mapa
    function deleteMarker(reportId) {
      if (reportMarkers[reportId]) {
        map.removeLayer(reportMarkers[reportId]);
        delete reportMarkers[reportId];
      }
    }

    // Crea el contenido HTML del popup
    function createPopupContent(report) {
        const imageHtml = report.imageData ? `<img src="${report.imageData}" class="photo-preview" style="max-width: 150px; border-radius: 8px;">` : '';
        return `
            <div class="popup-content">
                <b>Reporte No. ${report.id}</b><br/>
                <b>Tipo:</b> ${report.type === 'fuga' ? 'Fuga de Agua' : 'Falla en Infraestructura'}<br/>
                <b>Descripción:</b> ${escapeHtml(report.description)}<br/>
                ${imageHtml}
                <br/><br/>
                <button id="edit-${report.id}">✏️ Editar</button>
                <button id="delete-${report.id}">🗑️ Eliminar</button>
            </div>
        `;
    }

    // Escapa texto para evitar inyección HTML
    function escapeHtml(text) {
      return text
        .replace(/&/g, "&amp;")
        .replace(/</g, "&lt;")
        .replace(/>/g, "&gt;")
        .replace(/"/g, "&quot;")
        .replace(/'/g, "&#039;");
    }

    // Maneja el clic en el mapa para crear un nuevo reporte
    map.on("click", (e) => {
      currentLatLng = e.latlng;
      editingReportId = null;
      document.getElementById("reportType").value = "fuga";
      document.getElementById("description").value = "";
      imagePreview.style.display = 'none';
      uploadedFile = null;
      reportForm.style.display = "block";
    });

    // Vista previa de la imagen seleccionada
    photoUploadInput.addEventListener('change', (e) => {
        uploadedFile = e.target.files[0];
        if (uploadedFile) {
            const reader = new FileReader();
            reader.onload = (event) => {
                imagePreview.src = event.target.result;
                imagePreview.style.display = 'block';
            };
            reader.readAsDataURL(uploadedFile);
        }
    });

    // Guarda o actualiza un reporte en Firestore
    document.getElementById("saveBtn").addEventListener("click", async () => {
      const type = document.getElementById("reportType").value;
      const description = document.getElementById("description").value.trim();

      if (!description) {
        showMessage("Por favor, describe la incidencia.");
        return;
      }

      showSpinner();
      let imageData = null;
      if (uploadedFile) {
          try {
              imageData = await new Promise((resolve, reject) => {
                  const reader = new FileReader();
                  reader.onloadend = () => resolve(reader.result);
                  reader.onerror = reject;
                  reader.readAsDataURL(uploadedFile);
              });
          } catch (error) {
              console.error("Error al leer la imagen:", error);
              showMessage("Error al leer la imagen. El reporte se guardará sin foto.");
          }
      }

      const reportData = {
        type: type,
        description: description,
        location: {
          lat: currentLatLng.lat,
          lng: currentLatLng.lng,
        },
        imageData: imageData,
        timestamp: new Date().toISOString(),
        userId: userId
      };

      try {
        if (editingReportId) {
          const reportRef = doc(db, `artifacts/${appId}/public/data/reports`, editingReportId);
          await updateDoc(reportRef, reportData);
        } else {
          await addDoc(collection(db, `artifacts/${appId}/public/data/reports`), reportData);
        }
        hideSpinner();
        resetForm();
      } catch (error) {
        console.error("Error al guardar el reporte:", error);
        showMessage("Error al guardar el reporte. Revisa la consola.");
        hideSpinner();
      }
    });

    // Cancela el formulario de reporte
    document.getElementById("cancelBtn").addEventListener("click", () => {
      resetForm();
    });

    // Abre el formulario para editar un reporte
    async function editReport(reportId) {
        try {
            const reportRef = doc(db, `artifacts/${appId}/public/data/reports`, reportId);
            const docSnap = await getDoc(reportRef);

            if (docSnap.exists()) {
                const report = docSnap.data();
                editingReportId = reportId;
                currentLatLng = { lat: report.location.lat, lng: report.location.lng };
                document.getElementById("reportType").value = report.type;
                document.getElementById("description").value = report.description;
                
                if (report.imageData) {
                    imagePreview.src = report.imageData;
                    imagePreview.style.display = 'block';
                } else {
                    imagePreview.style.display = 'none';
                }
                
                uploadedFile = null;

                reportForm.style.display = "block";
            } else {
                showMessage("El reporte no existe.");
            }
        } catch (error) {
            console.error("Error al obtener el reporte para edición:", error);
            showMessage("Error al cargar los datos del reporte.");
        }
    }

    // Elimina un reporte de Firestore
    function deleteReport(reportId) {
      showMessage("¿Estás seguro de que deseas eliminar este reporte?");
      okBtn.textContent = "Sí";
      const oldOkOnClick = okBtn.onclick;
      okBtn.onclick = async () => {
        try {
          await deleteDoc(doc(db, `artifacts/${appId}/public/data/reports`, reportId));
          messageModal.style.display = 'none';
          okBtn.textContent = "OK";
          okBtn.onclick = oldOkOnClick;
        } catch (error) {
          console.error("Error al eliminar el reporte:", error);
          showMessage("Error al eliminar el reporte.");
        }
      };
    }

    // Reinicia el formulario y lo oculta
    function resetForm() {
      reportForm.style.display = "none";
      document.getElementById("reportType").value = "fuga";
      document.getElementById("description").value = "";
      photoUploadInput.value = '';
      imagePreview.style.display = 'none';
      uploadedFile = null;
      currentLatLng = null;
      editingReportId = null;
    }

    // Muestra la ubicación actual del usuario al cargar la página
    if (navigator.geolocation) {
      navigator.geolocation.getCurrentPosition(
        (pos) => {
          const lat = pos.coords.latitude;
          const lng = pos.coords.longitude;
          map.setView([lat, lng], 15);
          L.marker([lat, lng])
            .addTo(map)
            .bindPopup("📍 Aquí estás")
            .openPopup();
        },
        () => {
          console.warn("No se pudo obtener la ubicación");
        }
      );
    }
    
    // Inicia la aplicación
    initFirebase();
  </script>
</body>
</html>
