
import React, { useState, useEffect, useCallback, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged, signOut } from 'firebase/auth';
import { 
    getFirestore, doc, getDoc, setDoc, onSnapshot, collection, query, addDoc, updateDoc, deleteDoc
} from 'firebase/firestore';
import { LogOut, User, Users, Compass, DollarSign, Settings, Ticket, BarChart3, Plus, X, Pencil, Save, List, Search } from 'lucide-react';

// --- CONFIGURACIÓN DE FIREBASE Y GLOBALES ---
// Variables globales proporcionadas por el entorno de Canvas
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// HARDCODED ADMIN CREDENTIALS
const ADMIN_LOGIN = 'adm';
const ADMIN_CLAVE = 'florida439';

// Tasa de descuento para niños
const DESCUENTO_NINOS = 0.40;

// Definición de las colecciones de Firestore
const getPublicCollectionPath = (collectionName) => 
    `artifacts/${appId}/public/data/${collectionName}`;
const getConfigDocPath = (userId) => 
    `artifacts/${appId}/users/${userId}/config/app_settings`;

// --- UTILIDADES ---

/**
 * Convierte un enlace de YouTube estándar a un enlace incrustable (embed URL).
 * @param {string} url - Enlace de YouTube.
 * @returns {string} - URL de incrustación de YouTube.
 */
const getYoutubeEmbedUrl = (url) => {
    try {
        const urlObj = new URL(url);
        const videoId = urlObj.searchParams.get('v');
        if (videoId) {
            return `https://www.youtube.com/embed/${videoId}`;
        }
        // Manejar formatos de URL cortos (e.g., youtu.be/videoId)
        const match = url.match(/(?:youtu\.be\/|youtube\.com\/(?:embed\/|v\/|watch\?v=|watch\?feature=player_embedded&v=))([\w-]{10,12})/);
        return match ? `https://www.youtube.com/embed/${match[1]}` : null;
    } catch (e) {
        return null;
    }
};

/**
 * Función de retardo para manejar la espera en el login.
 * @param {number} ms - Milisegundos a esperar.
 */
const delay = (ms) => new Promise(resolve => setTimeout(resolve, ms));

// --- COMPONENTES UI REUTILIZABLES ---

const Button = ({ children, onClick, className = '', disabled = false, icon: Icon, color = 'blue' }) => (
    <button
        onClick={onClick}
        disabled={disabled}
        className={`
            flex items-center justify-center space-x-2 px-4 py-2 font-semibold rounded-lg shadow-md transition-all duration-200 
            ${disabled ? 'bg-gray-400 cursor-not-allowed' : 
             color === 'blue' ? 'bg-indigo-600 hover:bg-indigo-700 text-white' : 
             color === 'red' ? 'bg-red-600 hover:bg-red-700 text-white' :
             color === 'green' ? 'bg-green-600 hover:bg-green-700 text-white' :
             'bg-gray-200 hover:bg-gray-300 text-gray-800'}
            ${className}
        `}
    >
        {Icon && <Icon className="w-5 h-5" />}
        <span>{children}</span>
    </button>
);

const Input = ({ label, type = 'text', value, onChange, placeholder = '', name, required = false, className = '' }) => (
    <div className={`flex flex-col ${className}`}>
        <label className="text-sm font-medium text-gray-700 mb-1">{label}</label>
        <input
            type={type}
            value={value}
            onChange={onChange}
            placeholder={placeholder}
            name={name}
            required={required}
            className="px-3 py-2 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500 transition duration-150"
        />
    </div>
);

const Modal = ({ isOpen, onClose, title, children }) => {
    if (!isOpen) return null;
    return (
        <div className="fixed inset-0 bg-gray-600 bg-opacity-50 flex justify-center items-center z-50 p-4">
            <div className="bg-white rounded-xl shadow-2xl w-full max-w-lg p-6 transform transition-all duration-300 scale-100">
                <div className="flex justify-between items-center border-b pb-3 mb-4">
                    <h3 className="text-xl font-bold text-gray-800">{title}</h3>
                    <button onClick={onClose} className="text-gray-400 hover:text-gray-600 transition">
                        <X className="w-6 h-6" />
                    </button>
                </div>
                {children}
            </div>
        </div>
    );
};

// --- COMPONENTE PRINCIPAL DE LA APLICACIÓN ---

const App = () => {
    // --- ESTADO DE LA APLICACIÓN ---
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);
    const [currentUserId, setCurrentUserId] = useState(null);

    // Navegación y Roles
    const [page, setPage] = useState('home'); // 'home', 'admin_login', 'vendor_login', 'admin_menu', 'vendor_menu', ...
    const [role, setRole] = useState('none'); // 'none', 'admin', 'vendor'
    const [currentVendor, setCurrentVendor] = useState(null); // Datos del vendedor logueado

    // Datos de la Base de Datos
    const [vendedores, setVendedores] = useState([]);
    const [promotores, setPromotores] = useState([]);
    const [paseos, setPaseos] = useState([]);
    const [ventas, setVentas] = useState([]);
    const [vouchers, setVouchers] = useState([]);
    const [config, setConfig] = useState({
        appName: "Sistema de Gestión",
        logoUrl: "https://placehold.co/120x40/4f46e5/ffffff?text=LOGO",
        themeColor: "#4f46e5",
        voucherPolicy: "Política de Reintegro:\nLas cancelaciones deben solicitarse con 48 horas de anticipación. Se aplica una penalidad del 10%.",
        voucherStartNumber: 10000000,
    });
    
    // Estado de carga y errores
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);
    const [loginError, setLoginError] = useState('');

    // --- FIREBASE INICIALIZACIÓN Y AUTENTICACIÓN ---

    useEffect(() => {
        if (!Object.keys(firebaseConfig).length) {
            setError("Error: La configuración de Firebase no está disponible.");
            setLoading(false);
            return;
        }

        try {
            const app = initializeApp(firebaseConfig);
            const firestore = getFirestore(app);
            const authInstance = getAuth(app);
            
            setDb(firestore);
            setAuth(authInstance);

            // 1. Manejo de autenticación inicial (Custom Token o Anónimo)
            const signInUser = async () => {
                try {
                    if (initialAuthToken) {
                        await signInWithCustomToken(authInstance, initialAuthToken);
                    } else {
                        await signInAnonymously(authInstance);
                    }
                } catch (e) {
                    console.error("Error signing in:", e);
                    setError("Error de autenticación inicial. Intenta refrescar.");
                }
            };
            signInUser();

            // 2. Listener de cambio de estado de autenticación
            const unsubscribe = onAuthStateChanged(authInstance, (user) => {
                if (user) {
                    setCurrentUserId(user.uid);
                    // Si es el usuario anónimo generado por Canvas, asume 'home'
                    // Los roles se asignan al loguearse como Admin/Vendor
                } else {
                    setCurrentUserId(null);
                    setRole('none');
                }
                setIsAuthReady(true);
                setLoading(false);
            });

            return () => unsubscribe();
        } catch (e) {
            console.error("Error de inicialización de Firebase:", e);
            setError("Error al inicializar la base de datos.");
            setLoading(false);
        }
    }, []);

    // --- CARGA DE DATOS DE FIRESTORE (Real-time listeners) ---

    useEffect(() => {
        if (!db || !isAuthReady) return;

        // Listener de Configuración de la App
        const configRef = doc(db, getConfigDocPath(currentUserId || 'default_user'));
        const unsubConfig = onSnapshot(configRef, (docSnap) => {
            if (docSnap.exists()) {
                setConfig(prev => ({ ...prev, ...docSnap.data() }));
            }
        }, (e) => console.error("Error loading config:", e));

        // Listener de Vendedores
        const vendedoresQuery = query(collection(db, getPublicCollectionPath('vendedores')));
        const unsubVendedores = onSnapshot(vendedoresQuery, (snapshot) => {
            setVendedores(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
        }, (e) => console.error("Error loading vendedores:", e));

        // Listener de Promotores
        const promotoresQuery = query(collection(db, getPublicCollectionPath('promotores')));
        const unsubPromotores = onSnapshot(promotoresQuery, (snapshot) => {
            setPromotores(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
        }, (e) => console.error("Error loading promotores:", e));

        // Listener de Paseos
        const paseosQuery = query(collection(db, getPublicCollectionPath('paseos')));
        const unsubPaseos = onSnapshot(paseosQuery, (snapshot) => {
            setPaseos(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
        }, (e) => console.error("Error loading paseos:", e));

        // Listener de Ventas
        const ventasQuery = query(collection(db, getPublicCollectionPath('ventas')));
        const unsubVentas = onSnapshot(ventasQuery, (snapshot) => {
            setVentas(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
        }, (e) => console.error("Error loading ventas:", e));

        // Listener de Vouchers
        const vouchersQuery = query(collection(db, getPublicCollectionPath('vouchers')));
        const unsubVouchers = onSnapshot(vouchersQuery, (snapshot) => {
            setVouchers(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
        }, (e) => console.error("Error loading vouchers:", e));


        // Limpiar listeners al desmontar
        return () => {
            unsubConfig();
            unsubVendedores();
            unsubPromotores();
            unsubPaseos();
            unsubVentas();
            unsubVouchers();
        };

    }, [db, isAuthReady, currentUserId]);

    // --- FUNCIONES DE AUTENTICACIÓN Y NAVEGACIÓN ---

    const handleLogin = (login, clave, targetRole) => {
        setLoginError('');
        setLoading(true);

        // Retraso para simular proceso de login
        delay(500).then(() => {
            if (targetRole === 'admin') {
                if (login === ADMIN_LOGIN && clave === ADMIN_CLAVE) {
                    setRole('admin');
                    setPage('admin_menu');
                    setLoading(false);
                } else {
                    setLoginError('Credenciales de Administrador incorrectas.');
                    setLoading(false);
                }
            } else if (targetRole === 'vendor') {
                const vendor = vendedores.find(v => v.nombre.toLowerCase() === login.toLowerCase() && v.clave === clave);
                if (vendor) {
                    setRole('vendor');
                    setCurrentVendor(vendor);
                    setPage('vendor_menu');
                    setLoading(false);
                } else {
                    setLoginError('Credenciales de Vendedor incorrectas.');
                    setLoading(false);
                }
            }
        });
    };

    const handleLogout = () => {
        // En un entorno real, también llamarías a signOut(auth) si el usuario no es anónimo
        setRole('none');
        setCurrentVendor(null);
        setPage('home');
        setLoginError('');
    };

    const navigate = (newPage) => setPage(newPage);

    // --- COMPONENTE DE LOGIN ---

    const LoginForm = ({ roleType }) => {
        const [login, setLogin] = useState('');
        const [clave, setClave] = useState('');

        const handleSubmit = (e) => {
            e.preventDefault();
            handleLogin(login, clave, roleType);
        };

        return (
            <div className="max-w-md mx-auto p-6 bg-white rounded-xl shadow-lg mt-10">
                <h2 className="text-2xl font-bold text-gray-800 mb-6 text-center">
                    {roleType === 'admin' ? 'Login de Administrador' : 'Login de Vendedor'}
                </h2>
                <form onSubmit={handleSubmit} className="space-y-4">
                    <Input
                        label="Login"
                        type="text"
                        value={login}
                        onChange={(e) => setLogin(e.target.value)}
                        required
                    />
                    <Input
                        label="Clave"
                        type="password"
                        value={clave}
                        onChange={(e) => setClave(e.target.value)}
                        required
                    />
                    {loginError && <p className="text-red-500 text-sm">{loginError}</p>}
                    <Button 
                        type="submit" 
                        className="w-full" 
                        disabled={loading}
                        color="blue"
                    >
                        {loading ? 'Cargando...' : 'Ingresar'}
                    </Button>
                </form>
                <div className="mt-4 text-center">
                    <Button onClick={() => navigate('home')} color="gray" className="w-full">
                        Volver al Inicio
                    </Button>
                </div>
                {roleType === 'admin' && (
                    <p className="text-xs text-gray-400 mt-2 text-center">
                        Admin Login: {ADMIN_LOGIN} / Clave: {ADMIN_CLAVE}
                    </p>
                )}
            </div>
        );
    };

    // --- PANELES DE GESTIÓN (CRUD) ---

    // 1. Gestión de Vendedores (Admin)
    const VendedoresPanel = () => {
        const [editing, setEditing] = useState(null); // null, 'add', or vendor object
        const [deleting, setDeleting] = useState(null); // vendor object

        const initialFormState = { nombre: '', clave: '', telefono: '', email: '', porcentaje: 0 };
        const [formState, setFormState] = useState(initialFormState);
        const [formError, setFormError] = useState('');

        const handleEdit = (vendor) => {
            setFormState(vendor);
            setEditing(vendor);
        };

        const handleChange = (e) => {
            const { name, value } = e.target;
            setFormState(prev => ({ 
                ...prev, 
                [name]: name === 'porcentaje' ? (parseFloat(value) || 0) : value 
            }));
        };

        const saveVendedor = async (e) => {
            e.preventDefault();
            if (!db) return;
            setFormError('');

            // Validación simple
            if (!formState.nombre || !formState.clave) {
                setFormError("Nombre y Clave son obligatorios.");
                return;
            }

            try {
                const collectionRef = collection(db, getPublicCollectionPath('vendedores'));
                if (editing && editing !== 'add') {
                    // Actualizar
                    const docRef = doc(collectionRef, editing.id);
                    await updateDoc(docRef, { ...formState, porcentaje: formState.porcentaje });
                } else {
                    // Agregar
                    await addDoc(collectionRef, { 
                        ...formState, 
                        porcentaje: formState.porcentaje,
                        creator_id: currentUserId,
                        created_at: new Date()
                    });
                }
                setEditing(null);
                setFormState(initialFormState);
            } catch (e) {
                console.error("Error saving vendedor:", e);
                setFormError("Error al guardar el vendedor.");
            }
        };

        const deleteVendedor = async (vendor) => {
            if (!db || !vendor) return;
            try {
                const docRef = doc(db, getPublicCollectionPath('vendedores'), vendor.id);
                await deleteDoc(docRef);
                setDeleting(null);
            } catch (e) {
                console.error("Error deleting vendedor:", e);
                setFormError("Error al borrar el vendedor.");
            }
        };

        const renderForm = () => (
            <Modal isOpen={!!editing} onClose={() => { setEditing(null); setFormState(initialFormState); setFormError(''); }} 
                   title={editing === 'add' ? 'Agregar Nuevo Vendedor' : `Editar Vendedor: ${editing?.nombre}`}>
                <form onSubmit={saveVendedor} className="space-y-4">
                    <Input label="Nombre" name="nombre" value={formState.nombre} onChange={handleChange} required />
                    <Input label="Clave" name="clave" type="text" value={formState.clave} onChange={handleChange} required />
                    <Input label="Número de Teléfono" name="telefono" value={formState.telefono} onChange={handleChange} />
                    <Input label="Email" name="email" type="email" value={formState.email} onChange={handleChange} />
                    <Input label="Porcentaje de Comisión (%)" name="porcentaje" type="number" step="0.01" min="0" max="100" value={formState.porcentaje} onChange={handleChange} />
                    {formError && <p className="text-red-500 text-sm mt-2">{formError}</p>}
                    <Button type="submit" className="w-full" icon={Save}>
                        {editing === 'add' ? 'Salvar Nuevo Vendedor' : 'Salvar Cambios'}
                    </Button>
                </form>
            </Modal>
        );

        const renderDeleteModal = () => (
            <Modal isOpen={!!deleting} onClose={() => setDeleting(null)} title="Confirmar Borrado">
                <p className="text-lg mb-4">
                    ¿Está seguro de que desea **BORRAR** al vendedor **{deleting?.nombre}**?
                    Esta acción es irreversible.
                </p>
                <div className="flex justify-end space-x-4">
                    <Button onClick={() => setDeleting(null)} color="gray">Cancelar</Button>
                    <Button onClick={() => deleteVendedor(deleting)} color="red" icon={X}>Borrar Vendedor</Button>
                </div>
            </Modal>
        );

        return (
            <div className="space-y-6">
                <div className="flex justify-between items-center">
                    <h2 className="text-2xl font-bold">Gestión de Vendedores</h2>
                    <Button onClick={() => setEditing('add')} icon={Plus}>
                        Agregar Vendedor
                    </Button>
                </div>
                
                <div className="bg-white p-4 rounded-xl shadow-lg">
                    <h3 className="text-xl font-semibold mb-3 border-b pb-2">Lista de Vendedores</h3>
                    <div className="overflow-x-auto">
                        <table className="min-w-full divide-y divide-gray-200">
                            <thead className="bg-gray-50">
                                <tr>
                                    <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Nombre</th>
                                    <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Email</th>
                                    <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">% Comisión</th>
                                    <th className="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Acciones</th>
                                </tr>
                            </thead>
                            <tbody className="bg-white divide-y divide-gray-200">
                                {vendedores.map(vendor => (
                                    <tr key={vendor.id}>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900">{vendor.nombre}</td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">{vendor.email}</td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">{vendor.porcentaje}%</td>
                                        <td className="px-6 py-4 whitespace-nowrap text-right text-sm font-medium space-x-2">
                                            <Button onClick={() => handleEdit(vendor)} icon={Pencil} color="green" className="py-1 px-2 text-sm">Editar</Button>
                                            <Button onClick={() => setDeleting(vendor)} icon={X} color="red" className="py-1 px-2 text-sm">Borrar</Button>
                                        </td>
                                    </tr>
                                ))}
                            </tbody>
                        </table>
                    </div>
                </div>

                <Button onClick={() => navigate('admin_menu')} color="gray" className="w-full" icon={List}>
                    Volver al Menú Principal
                </Button>

                {renderForm()}
                {renderDeleteModal()}
            </div>
        );
    };
    
    // 2. Gestión de Promotores (Admin)
    const PromotoresPanel = () => {
        const [editing, setEditing] = useState(null); // null, 'add', or promoter object
        const [deleting, setDeleting] = useState(null); // promoter object
        const initialFormState = { nombre: '', porcentaje: 0 };
        const [formState, setFormState] = useState(initialFormState);
        const [formError, setFormError] = useState('');

        const handleEdit = (promotor) => {
            setFormState(promotor);
            setEditing(promotor);
        };

        const handleChange = (e) => {
            const { name, value } = e.target;
            setFormState(prev => ({ 
                ...prev, 
                [name]: name === 'porcentaje' ? (parseFloat(value) || 0) : value 
            }));
        };

        const savePromotor = async (e) => {
            e.preventDefault();
            if (!db) return;
            setFormError('');

            if (!formState.nombre) {
                setFormError("El Nombre es obligatorio.");
                return;
            }

            try {
                const collectionRef = collection(db, getPublicCollectionPath('promotores'));
                if (editing && editing !== 'add') {
                    const docRef = doc(collectionRef, editing.id);
                    await updateDoc(docRef, { ...formState, porcentaje: formState.porcentaje });
                } else {
                    await addDoc(collectionRef, { 
                        ...formState, 
                        porcentaje: formState.porcentaje,
                        creator_id: currentUserId,
                        created_at: new Date()
                    });
                }
                setEditing(null);
                setFormState(initialFormState);
            } catch (e) {
                console.error("Error saving promotor:", e);
                setFormError("Error al guardar el promotor.");
            }
        };

        const deletePromotor = async (promotor) => {
            if (!db || !promotor) return;
            try {
                const docRef = doc(db, getPublicCollectionPath('promotores'), promotor.id);
                await deleteDoc(docRef);
                setDeleting(null);
            } catch (e) {
                console.error("Error deleting promotor:", e);
                setFormError("Error al borrar el promotor.");
            }
        };

        const renderForm = () => (
            <Modal isOpen={!!editing} onClose={() => { setEditing(null); setFormState(initialFormState); setFormError(''); }} 
                   title={editing === 'add' ? 'Agregar Nuevo Promotor' : `Editar Promotor: ${editing?.nombre}`}>
                <form onSubmit={savePromotor} className="space-y-4">
                    <Input label="Nombre del Promotor" name="nombre" value={formState.nombre} onChange={handleChange} required />
                    <Input label="Porcentaje de Comisión (%)" name="porcentaje" type="number" step="0.01" min="0" max="100" value={formState.porcentaje} onChange={handleChange} />
                    {formError && <p className="text-red-500 text-sm mt-2">{formError}</p>}
                    <Button type="submit" className="w-full" icon={Save}>
                        {editing === 'add' ? 'Salvar Nuevo Promotor' : 'Salvar Cambios'}
                    </Button>
                </form>
            </Modal>
        );

        const renderDeleteModal = () => (
            <Modal isOpen={!!deleting} onClose={() => setDeleting(null)} title="Confirmar Exclusión">
                <p className="text-lg mb-4">
                    ¿Está seguro de que desea **EXCLUIR** al promotor **{deleting?.nombre}**?
                </p>
                <div className="flex justify-end space-x-4">
                    <Button onClick={() => setDeleting(null)} color="gray">Cancelar</Button>
                    <Button onClick={() => deletePromotor(deleting)} color="red" icon={X}>Excluir Promotor</Button>
                </div>
            </Modal>
        );

        return (
            <div className="space-y-6">
                <div className="flex justify-between items-center">
                    <h2 className="text-2xl font-bold">Gestión de Promotores</h2>
                    <Button onClick={() => setEditing('add')} icon={Plus}>
                        Agregar Promotor
                    </Button>
                </div>
                
                <div className="bg-white p-4 rounded-xl shadow-lg">
                    <h3 className="text-xl font-semibold mb-3 border-b pb-2">Lista de Promotores</h3>
                    <div className="overflow-x-auto">
                        <table className="min-w-full divide-y divide-gray-200">
                            <thead className="bg-gray-50">
                                <tr>
                                    <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Nombre</th>
                                    <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">% Comisión</th>
                                    <th className="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Acciones</th>
                                </tr>
                            </thead>
                            <tbody className="bg-white divide-y divide-gray-200">
                                {promotores.map(promotor => (
                                    <tr key={promotor.id}>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900">{promotor.nombre}</td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">{promotor.porcentaje}%</td>
                                        <td className="px-6 py-4 whitespace-nowrap text-right text-sm font-medium space-x-2">
                                            <Button onClick={() => handleEdit(promotor)} icon={Pencil} color="green" className="py-1 px-2 text-sm">Editar</Button>
                                            <Button onClick={() => setDeleting(promotor)} icon={X} color="red" className="py-1 px-2 text-sm">Excluir</Button>
                                        </td>
                                    </tr>
                                ))}
                            </tbody>
                        </table>
                    </div>
                </div>

                <Button onClick={() => navigate('admin_menu')} color="gray" className="w-full" icon={List}>
                    Volver al Menú Principal
                </Button>

                {renderForm()}
                {renderDeleteModal()}
            </div>
        );
    };

    // 3. Gestión de Paseos (Admin)
    const PaseosPanel = ({ readOnly = false, onSelectPaseo }) => {
        const [editing, setEditing] = useState(null);
        const initialFormState = { nombre: '', info: '', horarios: '', precio: 0, youtube_link: '', fotos: [''] };
        const [formState, setFormState] = useState(initialFormState);
        const [formError, setFormError] = useState('');
        
        const handleEdit = (paseo) => {
            setFormState({ ...paseo, fotos: paseo.fotos || [''] });
            setEditing(paseo);
        };

        const handleChange = (e) => {
            const { name, value } = e.target;
            setFormState(prev => ({ 
                ...prev, 
                [name]: name === 'precio' ? (parseFloat(value) || 0) : value 
            }));
        };
        
        const handleFotoChange = (index, value) => {
            const newFotos = [...formState.fotos];
            newFotos[index] = value;
            setFormState(prev => ({ ...prev, fotos: newFotos }));
        };

        const addFotoField = () => {
            setFormState(prev => ({ ...prev, fotos: [...prev.fotos, ''] }));
        };
        
        const removeFotoField = (index) => {
            const newFotos = formState.fotos.filter((_, i) => i !== index);
            setFormState(prev => ({ ...prev, fotos: newFotos.length > 0 ? newFotos : [''] }));
        };

        const savePaseo = async (e) => {
            e.preventDefault();
            if (!db) return;
            setFormError('');

            if (!formState.nombre || formState.precio <= 0) {
                setFormError("Nombre y Precio son obligatorios.");
                return;
            }

            try {
                const collectionRef = collection(db, getPublicCollectionPath('paseos'));
                // Limpiar fotos vacías
                const cleanFotos = formState.fotos.filter(f => f.trim() !== '');

                const dataToSave = { 
                    ...formState, 
                    precio: formState.precio,
                    fotos: cleanFotos,
                    // Asegurar que solo se guarde la parte incrustable del video
                    youtube_embed: getYoutubeEmbedUrl(formState.youtube_link)
                };

                if (editing && editing !== 'add') {
                    const docRef = doc(collectionRef, editing.id);
                    await updateDoc(docRef, dataToSave);
                } else {
                    await addDoc(collectionRef, { 
                        ...dataToSave,
                        creator_id: currentUserId,
                        created_at: new Date()
                    });
                }
                setEditing(null);
                setFormState(initialFormState);
            } catch (e) {
                console.error("Error saving paseo:", e);
                setFormError("Error al guardar el paseo.");
            }
        };
        
        const renderForm = () => (
            <Modal isOpen={!!editing} onClose={() => { setEditing(null); setFormState(initialFormState); setFormError(''); }} 
                   title={editing === 'add' ? 'Agregar Nuevo Paseo' : `Editar Paseo: ${editing?.nombre}`}>
                <form onSubmit={savePaseo} className="space-y-4">
                    <Input label="Nombre del Paseo" name="nombre" value={formState.nombre} onChange={handleChange} required />
                    
                    <div className="flex flex-col">
                        <label className="text-sm font-medium text-gray-700 mb-1">Informaciones del Paseo</label>
                        <textarea name="info" value={formState.info} onChange={handleChange} rows="3" className="px-3 py-2 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500"></textarea>
                    </div>

                    <Input label="Horarios del Paseo" name="horarios" value={formState.horarios} onChange={handleChange} placeholder="Ej: 8:00 a 16:00" />
                    <Input label="Precio del Paseo ($)" name="precio" type="number" step="0.01" min="0" value={formState.precio} onChange={handleChange} required />
                    <Input label="Link de YouTube (Video)" name="youtube_link" value={formState.youtube_link} onChange={handleChange} placeholder="https://www.youtube.com/watch?v=..." />
                    
                    <h4 className="text-lg font-semibold mt-6 mb-2">Cargar Fotos (URLs)</h4>
                    {formState.fotos.map((url, index) => (
                        <div key={index} className="flex space-x-2 items-center">
                            <Input 
                                label={`Foto ${index + 1}`} 
                                value={url} 
                                onChange={(e) => handleFotoChange(index, e.target.value)} 
                                placeholder="URL de la imagen" 
                                className="flex-grow"
                            />
                            <Button type="button" onClick={() => removeFotoField(index)} color="red" className="p-2 mt-5">
                                <X className="w-4 h-4" />
                            </Button>
                        </div>
                    ))}
                    <Button type="button" onClick={addFotoField} icon={Plus} color="gray">
                        Agregar Otra Foto
                    </Button>

                    {formError && <p className="text-red-500 text-sm mt-2">{formError}</p>}
                    <Button type="submit" className="w-full" icon={Save}>
                        Salvar Paseo
                    </Button>
                </form>
            </Modal>
        );

        return (
            <div className="space-y-6">
                <div className="flex justify-between items-center">
                    <h2 className="text-2xl font-bold">Lista de Paseos</h2>
                    {!readOnly && (
                        <Button onClick={() => setEditing('add')} icon={Plus}>
                            Agregar Paseo
                        </Button>
                    )}
                </div>
                
                <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                    {paseos.map(paseo => (
                        <div key={paseo.id} className="bg-white rounded-xl shadow-lg overflow-hidden transition-shadow duration-300 hover:shadow-xl">
                            <img 
                                src={paseo.fotos?.[0] || config.logoUrl} 
                                alt={paseo.nombre} 
                                className="w-full h-40 object-cover" 
                                onError={(e) => e.currentTarget.src = config.logoUrl} // Fallback
                            />
                            <div className="p-4">
                                <h3 className="text-xl font-bold text-gray-800 mb-2">{paseo.nombre}</h3>
                                <p className="text-2xl font-extrabold text-indigo-600 mb-2">${paseo.precio.toFixed(2)}</p>
                                <p className="text-sm text-gray-600 line-clamp-2">{paseo.info}</p>
                                
                                {paseo.youtube_embed && (
                                    <div className="mt-3">
                                        <h4 className="text-sm font-medium mb-1">Video (Vista Previa)</h4>
                                        <iframe
                                            className="w-full aspect-video rounded-lg"
                                            src={paseo.youtube_embed}
                                            title={`${paseo.nombre} video`}
                                            allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
                                            allowFullScreen
                                        ></iframe>
                                    </div>
                                )}

                                <div className="mt-4 flex justify-end space-x-2">
                                    {readOnly ? (
                                        <Button onClick={() => onSelectPaseo(paseo)} color="green" icon={Plus}>
                                            Seleccionar
                                        </Button>
                                    ) : (
                                        <Button onClick={() => handleEdit(paseo)} icon={Pencil} color="green">
                                            Editar Paseo
                                        </Button>
                                    )}
                                </div>
                            </div>
                        </div>
                    ))}
                </div>

                {!readOnly && (
                    <Button onClick={() => navigate('admin_menu')} color="gray" className="w-full" icon={List}>
                        Volver al Menú Principal
                    </Button>
                )}

                {renderForm()}
            </div>
        );
    };

    // 4. Gestión de Configuración (Editar Programa)
    const ConfiguracionPanel = () => {
        const [formState, setFormState] = useState(config);

        const handleChange = (e) => {
            const { name, value } = e.target;
            setFormState(prev => ({ 
                ...prev, 
                [name]: name === 'voucherStartNumber' ? (parseInt(value) || 0) : value 
            }));
        };

        const saveConfig = async (e) => {
            e.preventDefault();
            if (!db || !currentUserId) return;
            try {
                const configRef = doc(db, getConfigDocPath(currentUserId));
                await setDoc(configRef, formState, { merge: true });
                alert("Configuración salvada exitosamente.");
            } catch (e) {
                console.error("Error saving config:", e);
                alert("Error al guardar la configuración.");
            }
        };

        return (
            <div className="max-w-3xl mx-auto space-y-6 bg-white p-8 rounded-xl shadow-lg">
                <h2 className="text-2xl font-bold border-b pb-2">Editar Programa y Voucher</h2>
                <form onSubmit={saveConfig} className="space-y-6">
                    <h3 className="text-xl font-semibold text-indigo-600">Configuración del Programa</h3>
                    <Input label="Nombre del Programa" name="appName" value={formState.appName} onChange={handleChange} required />
                    <Input label="Logo del Programa (URL)" name="logoUrl" value={formState.logoUrl} onChange={handleChange} />
                    <Input label="Color Principal (Hex Code)" name="themeColor" type="color" value={formState.themeColor} onChange={handleChange} className="h-16" />

                    <h3 className="text-xl font-semibold text-indigo-600 pt-4 border-t">Configuración de Voucher</h3>
                    <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                        <Input 
                            label="Empezar Voucher desde el Número (8 dígitos min)" 
                            name="voucherStartNumber" 
                            type="number" 
                            min="10000000"
                            value={formState.voucherStartNumber} 
                            onChange={handleChange} 
                            required 
                        />
                        <div className="flex flex-col">
                            <label className="text-sm font-medium text-gray-700 mb-1">Política de Reintegro (Texto para el Voucher)</label>
                            <textarea 
                                name="voucherPolicy" 
                                value={formState.voucherPolicy} 
                                onChange={handleChange} 
                                rows="5" 
                                className="px-3 py-2 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500"
                            ></textarea>
                        </div>
                    </div>
                    
                    <Button type="submit" className="w-full" icon={Save}>
                        Salvar Cambios
                    </Button>
                </form>
                
                <Button onClick={() => navigate('admin_menu')} color="gray" className="w-full" icon={List}>
                    Volver al Menú Principal
                </Button>
            </div>
        );
    };

    // 5. Panel de Ventas (Admin)
    const VentasPanel = () => {
        const [startDate, setStartDate] = useState('');
        const [endDate, setEndDate] = useState('');

        const filteredVentas = useMemo(() => {
            return ventas.filter(venta => {
                if (!startDate && !endDate) return true;
                
                const saleDate = new Date(venta.fecha_venta).getTime();
                const start = startDate ? new Date(startDate).getTime() : 0;
                const end = endDate ? new Date(endDate).getTime() : new Date().getTime();

                return saleDate >= start && saleDate <= end;
            }).sort((a, b) => new Date(b.fecha_venta) - new Date(a.fecha_venta));
        }, [ventas, startDate, endDate]);

        const totalBalance = useMemo(() => {
            const balance = {
                totalVentas: 0,
                totalComisionVendedor: 0,
                totalComisionPromotor: 0,
                gananciaNeta: 0,
                vendedorComisiones: {},
                promotorComisiones: {}
            };

            filteredVentas.forEach(venta => {
                const gananciaEmpresa = venta.total_venta - venta.comision_vendedor - venta.comision_promotor;

                balance.totalVentas += venta.total_venta;
                balance.totalComisionVendedor += venta.comision_vendedor;
                balance.totalComisionPromotor += venta.comision_promotor;
                balance.gananciaNeta += gananciaEmpresa;

                // Comisiones por Vendedor
                const vendedor = vendedores.find(v => v.id === venta.vendedor_id)?.nombre || `ID: ${venta.vendedor_id}`;
                balance.vendedorComisiones[vendedor] = (balance.vendedorComisiones[vendedor] || 0) + venta.comision_vendedor;
                
                // Comisiones por Promotor
                const promotor = promotores.find(p => p.id === venta.promotor_id)?.nombre || `ID: ${venta.promotor_id}`;
                balance.promotorComisiones[promotor] = (balance.promotorComisiones[promotor] || 0) + venta.comision_promotor;
            });

            return balance;
        }, [filteredVentas, vendedores, promotores]);

        const printList = () => {
            const printContent = document.getElementById('print-area').innerHTML;
            const printWindow = window.open('', '', 'height=600,width=800');
            printWindow.document.write('<html><head><title>Lista de Ventas</title>');
            printWindow.document.write('<style>body{font-family:sans-serif;} table{width:100%; border-collapse:collapse;} th,td{border:1px solid #000; padding:8px; text-align:left;} h2,h3{text-align:center;}</style>');
            printWindow.document.write('</head><body>');
            printWindow.document.write(printContent);
            printWindow.document.write('</body></html>');
            printWindow.document.close();
            printWindow.print();
        };

        const printBalance = () => {
            const printContent = document.getElementById('balance-area').innerHTML;
            const printWindow = window.open('', '', 'height=600,width=800');
            printWindow.document.write('<html><head><title>Balance Semanal/General</title>');
            printWindow.document.write('<style>body{font-family:sans-serif;} .summary-card{border:1px solid #ccc; padding:15px; margin-bottom:10px; border-radius:8px;} .list{margin-top:20px;} table{width:100%; border-collapse:collapse;} th,td{border:1px solid #000; padding:8px; text-align:left;}</style>');
            printWindow.document.write('</head><body>');
            printWindow.document.write(printContent);
            printWindow.document.write('</body></html>');
            printWindow.document.close();
            printWindow.print();
        };

        return (
            <div className="space-y-6">
                <h2 className="text-2xl font-bold">Gestión de Ventas y Comisiones</h2>

                {/* Filtros de Búsqueda */}
                <div className="bg-white p-4 rounded-xl shadow-lg flex flex-col md:flex-row space-y-3 md:space-y-0 md:space-x-4">
                    <Input 
                        label="Fecha Inicial (dd-mm-aa)" 
                        type="date" 
                        value={startDate} 
                        onChange={(e) => setStartDate(e.target.value)} 
                        className="flex-grow"
                    />
                    <Input 
                        label="Fecha Final (dd-mm-aa)" 
                        type="date" 
                        value={endDate} 
                        onChange={(e) => setEndDate(e.target.value)} 
                        className="flex-grow"
                    />
                </div>
                
                {/* Balance e Impresión */}
                <div className="flex flex-col md:flex-row justify-between items-center space-y-3 md:space-y-0">
                    <div className="flex space-x-4">
                        <Button onClick={printList} icon={BarChart3} color="green">
                            Imprimir Lista de Ventas
                        </Button>
                        <Button onClick={printBalance} icon={BarChart3} color="red">
                            Imprimir Balance General
                        </Button>
                    </div>
                    <Button onClick={() => navigate('admin_menu')} color="gray" icon={List}>
                        Volver al Menú Principal
                    </Button>
                </div>

                {/* Área de Balance para Imprimir (Oculta) */}
                <div id="balance-area" className="hidden">
                    <h2>Balance General (Total: ${totalBalance.gananciaNeta.toFixed(2)})</h2>
                    <div className="summary-card">
                        <h3>Resumen Financiero</h3>
                        <p>Total Ventas: ${totalBalance.totalVentas.toFixed(2)}</p>
                        <p>Total Comisión Vendedor: ${totalBalance.totalComisionVendedor.toFixed(2)}</p>
                        <p>Total Comisión Promotor: ${totalBalance.totalComisionPromotor.toFixed(2)}</p>
                        <p>Ganancia Neta Empresa: ${totalBalance.gananciaNeta.toFixed(2)}</p>
                    </div>
                    <div className="list">
                        <h3>Comisiones por Vendedor</h3>
                        <table className="min-w-full">
                            <thead><tr><th>Vendedor</th><th>Comisión Total</th></tr></thead>
                            <tbody>
                                {Object.entries(totalBalance.vendedorComisiones).map(([name, com]) => (
                                    <tr key={name}><td>{name}</td><td>${com.toFixed(2)}</td></tr>
                                ))}
                            </tbody>
                        </table>
                    </div>
                    <div className="list">
                        <h3>Comisiones por Promotor</h3>
                        <table className="min-w-full">
                            <thead><tr><th>Promotor</th><th>Comisión Total</th></tr></thead>
                            <tbody>
                                {Object.entries(totalBalance.promotorComisiones).map(([name, com]) => (
                                    <tr key={name}><td>{name}</td><td>${com.toFixed(2)}</td></tr>
                                ))}
                            </tbody>
                        </table>
                    </div>
                </div>

                {/* Área de Ventas para Imprimir */}
                <div id="print-area" className="bg-white p-4 rounded-xl shadow-lg">
                    <h3 className="text-xl font-semibold mb-3 border-b pb-2">Lista Detallada de Ventas ({filteredVentas.length} Resultados)</h3>
                    <div className="overflow-x-auto">
                        <table className="min-w-full divide-y divide-gray-200">
                            <thead className="bg-gray-50">
                                <tr>
                                    <th className="px-3 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Vendedor</th>
                                    <th className="px-3 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Fecha</th>
                                    <th className="px-3 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Reserva</th>
                                    <th className="px-3 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Total Venta</th>
                                    <th className="px-3 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Com. Vendedor</th>
                                    <th className="px-3 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Com. Promotor</th>
                                </tr>
                            </thead>
                            <tbody className="bg-white divide-y divide-gray-200">
                                {filteredVentas.map(venta => {
                                    const vendedor = vendedores.find(v => v.id === venta.vendedor_id);
                                    return (
                                        <tr key={venta.id}>
                                            <td className="px-3 py-4 whitespace-nowrap text-sm">{vendedor?.nombre || 'N/A'}</td>
                                            <td className="px-3 py-4 whitespace-nowrap text-sm">{venta.fecha_venta}</td>
                                            <td className="px-3 py-4 whitespace-nowrap text-sm font-medium">{venta.reserva_nombre}</td>
                                            <td className="px-3 py-4 whitespace-nowrap text-sm text-right font-semibold">${venta.total_venta.toFixed(2)}</td>
                                            <td className="px-3 py-4 whitespace-nowrap text-sm text-right text-green-600">${venta.comision_vendedor.toFixed(2)}</td>
                                            <td className="px-3 py-4 whitespace-nowrap text-sm text-blue-600 text-right">${venta.comision_promotor.toFixed(2)}</td>
                                        </tr>
                                    );
                                })}
                            </tbody>
                        </table>
                    </div>
                </div>
            </div>
        );
    };

    // 6. Hacer Venta (Vendor)
    const HacerVentaPanel = () => {
        const [step, setStep] = useState(1); // 1: Seleccionar Paseos, 2: Detalles de la Reserva, 3: Finalizar
        const [selectedPaseos, setSelectedPaseos] = useState([]);
        const [reserva, setReserva] = useState({
            fecha: new Date().toISOString().split('T')[0],
            nombre: '',
            adultos: 1,
            ninos: 0,
            hotel: '',
            hora_inicio: '09:00',
            hora_fin: '17:00',
            promotorId: '',
        });
        const [ventaFinal, setVentaFinal] = useState(null);
        const [ventaError, setVentaError] = useState('');

        const handleSelectPaseo = (paseo) => {
            if (!selectedPaseos.find(p => p.id === paseo.id)) {
                setSelectedPaseos([...selectedPaseos, paseo]);
            } else {
                setSelectedPaseos(selectedPaseos.filter(p => p.id !== paseo.id));
            }
        };

        const handleReservaChange = (e) => {
            const { name, value } = e.target;
            setReserva(prev => ({ 
                ...prev, 
                [name]: name === 'adultos' || name === 'ninos' ? (parseInt(value) || 0) : value
            }));
        };

        const calcularTotal = useMemo(() => {
            let subtotal = selectedPaseos.reduce((sum, paseo) => sum + paseo.precio, 0);
            
            // Multiplicar por la cantidad de personas
            subtotal = subtotal * (reserva.adultos + reserva.ninos);
            
            // Aplicar descuento a niños
            const totalDescuentoNinos = selectedPaseos.reduce((sum, paseo) => sum + paseo.precio, 0) * reserva.ninos * DESCUENTO_NINOS;
            
            const total = subtotal - totalDescuentoNinos;
            return {
                subtotal: subtotal,
                descuento: totalDescuentoNinos,
                total: Math.max(0, total) // Evitar valores negativos
            };
        }, [selectedPaseos, reserva.adultos, reserva.ninos]);

        const finalizarVenta = async () => {
            if (!db || !currentVendor) return;
            setVentaError('');
            
            if (selectedPaseos.length === 0 || !reserva.nombre || !reserva.promotorId) {
                setVentaError("Debe seleccionar paseos, completar el nombre de la reserva y seleccionar un promotor.");
                return;
            }

            try {
                const totalCalculado = calcularTotal;
                const promotor = promotores.find(p => p.id === reserva.promotorId);

                if (!promotor) {
                    setVentaError("Promotor no válido.");
                    return;
                }

                // Cálculo de Comisiones
                const comisionVendedor = totalCalculado.total * (currentVendor.porcentaje / 100);
                const comisionPromotor = totalCalculado.total * (promotor.porcentaje / 100);

                const ventaData = {
                    vendedor_id: currentVendor.id,
                    promotor_id: reserva.promotorId,
                    reserva_nombre: reserva.nombre,
                    fecha_venta: new Date().toISOString().split('T')[0],
                    fecha_paseo: reserva.fecha,
                    hotel: reserva.hotel,
                    hora_inicio: reserva.hora_inicio,
                    hora_fin: reserva.hora_fin,
                    adultos: reserva.adultos,
                    ninos: reserva.ninos,
                    total_venta: totalCalculado.total,
                    comision_vendedor: comisionVendedor,
                    comision_promotor: comisionPromotor,
                    paseos_ids: selectedPaseos.map(p => p.id),
                    paseos_nombres: selectedPaseos.map(p => p.nombre),
                    subtotal: totalCalculado.subtotal,
                    descuento_ninos: totalCalculado.descuento,
                };
                
                const collectionRef = collection(db, getPublicCollectionPath('ventas'));
                const newVentaRef = await addDoc(collectionRef, ventaData);

                // Generar Voucher (Obtener el próximo número)
                const nextVoucherNumber = vouchers.length > 0 
                    ? Math.max(...vouchers.map(v => v.voucher_number), config.voucherStartNumber) + 1
                    : config.voucherStartNumber;

                const voucherData = {
                    venta_id: newVentaRef.id,
                    vendedor_id: currentVendor.id,
                    voucher_number: nextVoucherNumber,
                    fecha_generacion: new Date().toISOString().split('T')[0],
                    reserva_nombre: reserva.nombre,
                    paseos_nombres: ventaData.paseos_nombres.join(', '),
                };
                await addDoc(collection(db, getPublicCollectionPath('vouchers')), voucherData);

                setVentaFinal({ ...ventaData, voucher_number: nextVoucherNumber });
                setStep(3); // Ir a la pantalla de finalización/impresión

            } catch (e) {
                console.error("Error al finalizar la venta:", e);
                setVentaError("Ocurrió un error al procesar la venta.");
            }
        };

        const resetVenta = () => {
            setSelectedPaseos([]);
            setReserva({
                fecha: new Date().toISOString().split('T')[0],
                nombre: '', adultos: 1, ninos: 0, hotel: '', hora_inicio: '09:00', hora_fin: '17:00', promotorId: '',
            });
            setVentaFinal(null);
            setStep(1);
            setVentaError('');
        };

        const printVoucher = (ventaData) => {
            const voucherHtml = `
                <div style="font-family: Arial, sans-serif; padding: 20px; border: 2px solid #333; max-width: 800px; margin: 0 auto;">
                    <div style="display: flex; justify-content: space-between; align-items: center; border-bottom: 2px solid #333; padding-bottom: 10px; margin-bottom: 20px;">
                        <img src="${config.logoUrl}" alt="${config.appName}" style="max-width: 150px; height: auto;"/>
                        <h1 style="color: #4f46e5; margin: 0; font-size: 28px;">${config.appName}</h1>
                    </div>
                    
                    <h2 style="font-size: 24px; text-align: center; margin-bottom: 20px;">VOUCHER DE RESERVA #${ventaData.voucher_number}</h2>
                    
                    <div style="border: 1px solid #ddd; padding: 15px; margin-bottom: 20px; border-radius: 8px;">
                        <p><strong>Cliente:</strong> ${ventaData.reserva_nombre}</p>
                        <p><strong>Paseos:</strong> ${ventaData.paseos_nombres.join(', ')}</p>
                        <p><strong>Fecha del Paseo:</strong> ${ventaData.fecha_paseo} | <strong>Horario:</strong> ${ventaData.hora_inicio} a ${ventaData.hora_fin}</p>
                        <p><strong>Adultos:</strong> ${ventaData.adultos} | <strong>Niños:</strong> ${ventaData.ninos}</p>
                        <p><strong>Hotel:</strong> ${ventaData.hotel}</p>
                    </div>

                    <table style="width: 100%; border-collapse: collapse; margin-bottom: 20px;">
                        <tr><td style="border: 1px solid #ddd; padding: 8px;">SUBTOTAL</td><td style="border: 1px solid #ddd; padding: 8px; text-align: right;">$${ventaData.subtotal.toFixed(2)}</td></tr>
                        <tr><td style="border: 1px solid #ddd; padding: 8px;">DESCUENTO NIÑOS (40%)</td><td style="border: 1px solid #ddd; padding: 8px; text-align: right; color: red;">-$${ventaData.descuento_ninos.toFixed(2)}</td></tr>
                        <tr style="font-size: 18px; font-weight: bold;"><td style="border: 1px solid #ddd; padding: 10px;">TOTAL PAGADO</td><td style="border: 1px solid #ddd; padding: 10px; text-align: right; color: #4f46e5;">$${ventaData.total_venta.toFixed(2)}</td></tr>
                    </table>
                    
                    <div style="margin-top: 30px; border-top: 1px solid #eee; padding-top: 20px;">
                        <h3 style="font-size: 16px; margin-bottom: 10px;">POLÍTICA DE REINTEGRO Y CANCELACIÓN</h3>
                        <pre style="white-space: pre-wrap; font-size: 12px; color: #555;">${config.voucherPolicy}</pre>
                    </div>
                    <p style="text-align: right; font-size: 10px; margin-top: 10px;">Vendedor: ${currentVendor.nombre}</p>
                </div>
            `;
            
            const printWindow = window.open('', '', 'height=600,width=800');
            printWindow.document.write('<html><head><title>Voucher de Reserva</title></head><body>');
            printWindow.document.write(voucherHtml);
            printWindow.document.write('</body></html>');
            printWindow.document.close();
            printWindow.print();
        };

        if (step === 1) {
            return (
                <div className="space-y-6">
                    <h2 className="text-2xl font-bold">Paso 1: Seleccionar Paseos</h2>
                    <div className="flex flex-wrap gap-3">
                        {paseos.map(p => (
                            <div key={p.id} className={`p-3 border-2 rounded-lg shadow-md cursor-pointer transition-all ${selectedPaseos.find(sp => sp.id === p.id) ? 'border-green-500 bg-green-50' : 'border-gray-200 bg-white hover:bg-gray-50'}`}
                                 onClick={() => handleSelectPaseo(p)}>
                                <h4 className="font-semibold">{p.nombre}</h4>
                                <p className="text-sm text-indigo-600">${p.precio.toFixed(2)}</p>
                            </div>
                        ))}
                    </div>
                    <div className="flex justify-between items-center pt-4 border-t">
                        <Button onClick={() => navigate('vendor_menu')} color="gray" icon={List}>Volver al Menú</Button>
                        <Button onClick={() => setStep(2)} disabled={selectedPaseos.length === 0} icon={Plus}>Continuar ({selectedPaseos.length} Paseos)</Button>
                    </div>
                </div>
            );
        }

        if (step === 2) {
            return (
                <div className="space-y-6">
                    <h2 className="text-2xl font-bold">Paso 2: Detalles de la Reserva</h2>
                    
                    <div className="bg-white p-6 rounded-xl shadow-lg space-y-4">
                        <h3 className="text-xl font-semibold mb-3">Información del Cliente y Paseo</h3>
                        <Input label="Nombre de la Reserva" name="nombre" value={reserva.nombre} onChange={handleReservaChange} required />
                        
                        <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                            <Input label="Fecha del Paseo" name="fecha" type="date" value={reserva.fecha} onChange={handleReservaChange} required />
                            <Input label="Hotel de Hospedaje" name="hotel" value={reserva.hotel} onChange={handleReservaChange} placeholder="Nombre del Hotel" />
                        </div>

                        <div className="grid grid-cols-2 gap-4">
                            <Input label="Horario de Inicio (XXXX)" name="hora_inicio" value={reserva.hora_inicio} onChange={handleReservaChange} type="time" />
                            <Input label="Horario de Fin (XXXX)" name="hora_fin" value={reserva.hora_fin} onChange={handleReservaChange} type="time" />
                        </div>
                        
                        <div className="grid grid-cols-2 gap-4">
                            <Input label="Cantidad de Adultos" name="adultos" type="number" min="1" value={reserva.adultos} onChange={handleReservaChange} required />
                            <Input label="Cantidad de Niños" name="ninos" type="number" min="0" value={reserva.ninos} onChange={handleReservaChange} />
                        </div>
                        
                        <div className="flex flex-col">
                            <label className="text-sm font-medium text-gray-700 mb-1">Seleccionar Promotor</label>
                            <select name="promotorId" value={reserva.promotorId} onChange={handleReservaChange} required 
                                    className="px-3 py-2 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500 transition duration-150">
                                <option value="">--- Seleccione un Promotor ---</option>
                                {promotores.map(p => (
                                    <option key={p.id} value={p.id}>{p.nombre} ({p.porcentaje}%)</option>
                                ))}
                            </select>
                        </div>
                    </div>

                    <div className="bg-indigo-50 p-4 rounded-xl shadow-inner space-y-2">
                        <h3 className="text-xl font-bold text-indigo-800">Cálculo Total</h3>
                        <p>Paseos Seleccionados: <span className="font-semibold">{selectedPaseos.map(p => p.nombre).join(', ')}</span></p>
                        <p>Subtotal (Sin Descuento): <span className="font-semibold text-gray-700">${calcularTotal.subtotal.toFixed(2)}</span></p>
                        <p>Descuento Niños ({DESCUENTO_NINOS * 100}%): <span className="font-semibold text-red-500">-${calcularTotal.descuento.toFixed(2)}</span></p>
                        <p className="text-2xl font-extrabold text-indigo-700 pt-2 border-t mt-2">TOTAL A PAGAR: <span className="float-right">${calcularTotal.total.toFixed(2)}</span></p>
                    </div>
                    
                    {ventaError && <p className="text-red-500 font-medium">{ventaError}</p>}
                    
                    <div className="flex justify-between items-center pt-4 border-t">
                        <Button onClick={() => setStep(1)} color="gray" icon={List}>Volver a Paseos</Button>
                        <Button onClick={finalizarVenta} icon={DollarSign}>Finalizar Venta</Button>
                    </div>
                </div>
            );
        }

        if (step === 3) {
            return (
                <div className="space-y-6 text-center">
                    <h2 className="text-3xl font-bold text-green-600">¡Venta Finalizada con Éxito!</h2>
                    <p className="text-xl">Voucher Generado: **#{ventaFinal?.voucher_number}**</p>
                    <p>Total de Venta: **${ventaFinal?.total_venta.toFixed(2)}**</p>
                    
                    <div className="flex justify-center space-x-4 pt-4">
                        <Button onClick={() => printVoucher(ventaFinal)} icon={Ticket}>
                            Imprimir Voucher
                        </Button>
                        <Button onClick={resetVenta} color="green" icon={Plus}>
                            Hacer Otra Venta
                        </Button>
                    </div>
                    <Button onClick={() => navigate('vendor_menu')} color="gray" className="w-full" icon={List}>
                        Volver al Menú Principal
                    </Button>
                </div>
            );
        }

        return null;
    };

    // 7. Panel de Vouchers (Vendor)
    const VendedorVouchersPanel = () => {
        const vendorVouchers = vouchers.filter(v => v.vendedor_id === currentVendor?.id);

        const getVentaDetails = (ventaId) => {
            const venta = ventas.find(v => v.id === ventaId);
            if (!venta) return null;
            return venta;
        };

        const printAndDownloadVoucher = (voucher) => {
            const venta = getVentaDetails(voucher.venta_id);
            if (!venta) return alert("Detalles de venta no encontrados.");

            const ventaData = {
                ...venta,
                voucher_number: voucher.voucher_number
            };

            // 1. Imprimir
            printVoucher(ventaData);

            // 2. Simular descarga PDF (el navegador solo permite guardar HTML o TXT en este entorno)
            const voucherText = `VOUCHER #${voucher.voucher_number}\nCliente: ${venta.reserva_nombre}\nPaseos: ${venta.paseos_nombres.join(', ')}\nTotal: $${venta.total_venta.toFixed(2)}\nPolítica: ${config.voucherPolicy}`;
            const blob = new Blob([voucherText], { type: 'text/plain' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = `Voucher_${voucher.voucher_number}.txt`; // Simular PDF
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
            URL.revokeObjectURL(url);
            alert("Voucher impreso y descarga de texto (simulando PDF) iniciada.");
        };

        const printVoucher = (ventaData) => {
            // Reusa la función de impresión de HacerVentaPanel
            // Nota: Aquí se está redefiniendo solo para que el componente sea autocontenido, 
            // pero en React real se pasaría como prop o se definiría en el contexto.
            const voucherHtml = `
                <div style="font-family: Arial, sans-serif; padding: 20px; border: 2px solid #333; max-width: 800px; margin: 0 auto;">
                    <div style="display: flex; justify-content: space-between; align-items: center; border-bottom: 2px solid #333; padding-bottom: 10px; margin-bottom: 20px;">
                        <img src="${config.logoUrl}" alt="${config.appName}" style="max-width: 150px; height: auto;"/>
                        <h1 style="color: #4f46e5; margin: 0; font-size: 28px;">${config.appName}</h1>
                    </div>
                    <h2 style="font-size: 24px; text-align: center; margin-bottom: 20px;">VOUCHER DE RESERVA #${ventaData.voucher_number}</h2>
                    <div style="border: 1px solid #ddd; padding: 15px; margin-bottom: 20px; border-radius: 8px;">
                        <p><strong>Cliente:</strong> ${ventaData.reserva_nombre}</p>
                        <p><strong>Paseos:</strong> ${ventaData.paseos_nombres.join(', ')}</p>
                        <p><strong>Fecha del Paseo:</strong> ${ventaData.fecha_paseo} | <strong>Horario:</strong> ${ventaData.hora_inicio} a ${ventaData.hora_fin}</p>
                        <p><strong>Adultos:</strong> ${ventaData.adultos} | <strong>Niños:</strong> ${ventaData.ninos}</p>
                        <p><strong>Hotel:</strong> ${ventaData.hotel}</p>
                    </div>
                    <table style="width: 100%; border-collapse: collapse; margin-bottom: 20px;">
                        <tr style="font-size: 18px; font-weight: bold;"><td style="border: 1px solid #ddd; padding: 10px;">TOTAL PAGADO</td><td style="border: 1px solid #ddd; padding: 10px; text-align: right; color: #4f46e5;">$${ventaData.total_venta.toFixed(2)}</td></tr>
                    </table>
                    <div style="margin-top: 30px; border-top: 1px solid #eee; padding-top: 20px;">
                        <h3 style="font-size: 16px; margin-bottom: 10px;">POLÍTICA DE REINTEGRO Y CANCELACIÓN</h3>
                        <pre style="white-space: pre-wrap; font-size: 12px; color: #555;">${config.voucherPolicy}</pre>
                    </div>
                    <p style="text-align: right; font-size: 10px; margin-top: 10px;">Vendedor: ${currentVendor.nombre}</p>
                </div>
            `;
            
            const printWindow = window.open('', '', 'height=600,width=800');
            printWindow.document.write('<html><head><title>Voucher de Reserva</title></head><body>');
            printWindow.document.write(voucherHtml);
            printWindow.document.write('</body></html>');
            printWindow.document.close();
            printWindow.print();
        };

        return (
            <div className="space-y-6">
                <h2 className="text-2xl font-bold">Mis Vouchers Generados</h2>
                
                <div className="bg-white p-4 rounded-xl shadow-lg">
                    <h3 className="text-xl font-semibold mb-3 border-b pb-2">Lista de Vouchers</h3>
                    <div className="overflow-x-auto">
                        <table className="min-w-full divide-y divide-gray-200">
                            <thead className="bg-gray-50">
                                <tr>
                                    <th className="px-3 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Voucher #</th>
                                    <th className="px-3 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Cliente</th>
                                    <th className="px-3 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Fecha Venta</th>
                                    <th className="px-3 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Acciones</th>
                                </tr>
                            </thead>
                            <tbody className="bg-white divide-y divide-gray-200">
                                {vendorVouchers.map(voucher => (
                                    <tr key={voucher.id}>
                                        <td className="px-3 py-4 whitespace-nowrap text-sm font-bold text-indigo-600">#{voucher.voucher_number}</td>
                                        <td className="px-3 py-4 whitespace-nowrap text-sm">{voucher.reserva_nombre}</td>
                                        <td className="px-3 py-4 whitespace-nowrap text-sm">{voucher.fecha_generacion}</td>
                                        <td className="px-3 py-4 whitespace-nowrap text-right text-sm font-medium">
                                            <Button onClick={() => printAndDownloadVoucher(voucher)} icon={Ticket} color="blue" className="py-1 px-2 text-sm">Imprimir / Descargar</Button>
                                        </td>
                                    </tr>
                                ))}
                            </tbody>
                        </table>
                    </div>
                </div>

                <Button onClick={() => navigate('vendor_menu')} color="gray" className="w-full" icon={List}>
                    Volver al Menú Principal
                </Button>
            </div>
        );
    };

    // 8. Historial de Ventas (Vendor)
    const VendedorHistorialPanel = () => {
        const [startDate, setStartDate] = useState('');
        const [endDate, setEndDate] = useState('');
        
        const vendorVentas = ventas.filter(v => v.vendedor_id === currentVendor?.id);

        const filteredVentas = useMemo(() => {
            return vendorVentas.filter(venta => {
                if (!startDate && !endDate) return true;
                
                const saleDate = new Date(venta.fecha_venta).getTime();
                const start = startDate ? new Date(startDate).getTime() : 0;
                const end = endDate ? new Date(endDate).getTime() : new Date().getTime();

                return saleDate >= start && saleDate <= end;
            }).sort((a, b) => new Date(b.fecha_venta) - new Date(a.fecha_venta));
        }, [vendorVentas, startDate, endDate]);

        const totalComision = filteredVentas.reduce((sum, venta) => sum + venta.comision_vendedor, 0);

        return (
            <div className="space-y-6">
                <h2 className="text-2xl font-bold">Mi Historial de Ventas</h2>

                {/* Filtros */}
                <div className="bg-white p-4 rounded-xl shadow-lg flex flex-col md:flex-row space-y-3 md:space-y-0 md:space-x-4">
                    <Input 
                        label="Fecha Inicial (dd-mm-aa)" 
                        type="date" 
                        value={startDate} 
                        onChange={(e) => setStartDate(e.target.value)} 
                        className="flex-grow"
                    />
                    <Input 
                        label="Fecha Final (dd-mm-aa)" 
                        type="date" 
                        value={endDate} 
                        onChange={(e) => setEndDate(e.target.value)} 
                        className="flex-grow"
                    />
                </div>

                <div className="bg-indigo-50 p-4 rounded-xl shadow-inner">
                    <h3 className="text-xl font-bold text-indigo-700">Comisión Total Acumulada: ${totalComision.toFixed(2)}</h3>
                </div>

                {/* Lista de Ventas */}
                <div className="bg-white p-4 rounded-xl shadow-lg">
                    <h3 className="text-xl font-semibold mb-3 border-b pb-2">Ventas Realizadas ({filteredVentas.length} Resultados)</h3>
                    <div className="overflow-x-auto">
                        <table className="min-w-full divide-y divide-gray-200">
                            <thead className="bg-gray-50">
                                <tr>
                                    <th className="px-3 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Fecha Venta</th>
                                    <th className="px-3 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Reserva</th>
                                    <th className="px-3 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Total Venta</th>
                                    <th className="px-3 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Mi Comisión</th>
                                </tr>
                            </thead>
                            <tbody className="bg-white divide-y divide-gray-200">
                                {filteredVentas.map(venta => (
                                    <tr key={venta.id}>
                                        <td className="px-3 py-4 whitespace-nowrap text-sm">{venta.fecha_venta}</td>
                                        <td className="px-3 py-4 whitespace-nowrap text-sm font-medium">{venta.reserva_nombre}</td>
                                        <td className="px-3 py-4 whitespace-nowrap text-sm text-right font-semibold">${venta.total_venta.toFixed(2)}</td>
                                        <td className="px-3 py-4 whitespace-nowrap text-sm text-right text-green-600 font-bold">${venta.comision_vendedor.toFixed(2)}</td>
                                    </tr>
                                ))}
                            </tbody>
                        </table>
                    </div>
                </div>

                <Button onClick={() => navigate('vendor_menu')} color="gray" className="w-full" icon={List}>
                    Volver al Menú Principal
                </Button>
            </div>
        );
    };

    // --- MENÚS PRINCIPALES ---

    const AdminMenu = () => (
        <div className="max-w-4xl mx-auto space-y-6">
            <h2 className="text-3xl font-bold text-gray-800 text-center">Menú de Administrador</h2>
            <div className="grid grid-cols-2 md:grid-cols-3 gap-6">
                <Button onClick={() => navigate('admin_vendedores')} icon={Users} className="h-24 text-xl">
                    Gestión de Vendedores
                </Button>
                <Button onClick={() => navigate('admin_promotores')} icon={User} className="h-24 text-xl">
                    Gestión de Promotores
                </Button>
                <Button onClick={() => navigate('admin_paseos')} icon={Compass} className="h-24 text-xl">
                    Gestión de Paseos
                </Button>
                <Button onClick={() => navigate('admin_ventas')} icon={DollarSign} className="h-24 text-xl">
                    Reporte de Ventas
                </Button>
                <Button onClick={() => navigate('admin_config')} icon={Settings} className="h-24 text-xl">
                    Editar Programa/Voucher
                </Button>
                <Button onClick={handleLogout} icon={LogOut} color="red" className="h-24 text-xl">
                    Cerrar Sesión
                </Button>
            </div>
        </div>
    );

    const VendorMenu = () => (
        <div className="max-w-4xl mx-auto space-y-6">
            <h2 className="text-3xl font-bold text-gray-800 text-center">
                Menú de Vendedor: {currentVendor?.nombre}
            </h2>
            <div className="grid grid-cols-2 md:grid-cols-3 gap-6">
                <Button onClick={() => navigate('vendor_paseos')} icon={Compass} className="h-24 text-xl">
                    Ver Paseos
                </Button>
                <Button onClick={() => navigate('vendor_hacer_venta')} icon={DollarSign} className="h-24 text-xl">
                    Hacer Venta
                </Button>
                <Button onClick={() => navigate('vendor_vouchers')} icon={Ticket} className="h-24 text-xl">
                    Mis Vouchers
                </Button>
                <Button onClick={() => navigate('vendor_historial')} icon={BarChart3} className="h-24 text-xl">
                    Historial de Ventas
                </Button>
                <Button onClick={handleLogout} icon={LogOut} color="red" className="h-24 text-xl col-span-2 md:col-span-1">
                    Cerrar Sesión
                </Button>
            </div>
        </div>
    );

    // --- RENDERIZADO PRINCIPAL ---

    const renderPage = () => {
        if (loading) {
            return <div className="text-center text-xl font-semibold mt-20">Cargando datos del sistema...</div>;
        }
        if (error) {
            return <div className="text-center text-red-500 text-xl font-semibold mt-20">{error}</div>;
        }

        switch (page) {
            case 'home':
                return (
                    <div className="flex flex-col items-center justify-center min-h-[400px] space-y-8 p-4">
                        <h1 className="text-4xl font-extrabold text-gray-900">Bienvenido al {config.appName}</h1>
                        <p className="text-lg text-gray-600">Selecciona tu rol para ingresar:</p>
                        <div className="flex space-x-6">
                            <Button onClick={() => navigate('vendor_login')} icon={Users} className="py-3 px-8 text-xl">
                                Vendedor
                            </Button>
                            <Button onClick={() => navigate('admin_login')} icon={Settings} className="py-3 px-8 text-xl">
                                Administrador
                            </Button>
                        </div>
                        <p className="text-xs text-gray-400 mt-2">ID de App: {appId}</p>
                    </div>
                );
            case 'admin_login':
                return <LoginForm roleType="admin" />;
            case 'vendor_login':
                return <LoginForm roleType="vendor" />;
            case 'admin_menu':
                return <AdminMenu />;
            case 'vendor_menu':
                return <VendorMenu />;
            
            // ADMIN PANELS
            case 'admin_vendedores':
                return <VendedoresPanel />;
            case 'admin_promotores':
                return <PromotoresPanel />;
            case 'admin_paseos':
                return <PaseosPanel readOnly={false} />;
            case 'admin_ventas':
                return <VentasPanel />;
            case 'admin_config':
                return <ConfiguracionPanel />;
            
            // VENDOR PANELS
            case 'vendor_paseos':
                return <PaseosPanel readOnly={true} onSelectPaseo={() => {}} />;
            case 'vendor_hacer_venta':
                return <HacerVentaPanel />;
            case 'vendor_vouchers':
                return <VendedorVouchersPanel />;
            case 'vendor_historial':
                return <VendedorHistorialPanel />;
                
            default:
                return <div className="text-center mt-20">Página no encontrada.</div>;
        }
    };

    return (
        <div className="min-h-screen bg-gray-100 p-4 sm:p-8">
            <header className="mb-8 p-4 bg-white shadow-md rounded-xl flex justify-between items-center" style={{ borderLeft: `8px solid ${config.themeColor}` }}>
                <div className="flex items-center space-x-4">
                    <img src={config.logoUrl} alt="Logo" className="h-10" onError={(e) => e.currentTarget.src = 'https://placehold.co/120x40/4f46e5/ffffff?text=LOGO'}/>
                    <h1 className="text-2xl font-bold text-gray-800 hidden sm:block">{config.appName}</h1>
                </div>
                {role !== 'none' && (
                    <div className="text-right">
                        <p className="text-sm font-semibold text-gray-700">
                            {role === 'admin' ? 'Administrador' : currentVendor?.nombre}
                        </p>
                        <p className="text-xs text-gray-500">{currentVendor?.email || 'N/A'}</p>
                    </div>
                )}
            </header>
            
            <main>
                <div className="max-w-7xl mx-auto">
                    {renderPage()}
                </div>
            </main>
        </div>
    );
};

export default App;
