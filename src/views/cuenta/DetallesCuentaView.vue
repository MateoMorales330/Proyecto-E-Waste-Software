<template>
  <cuenta-shell activo="detalles" titulo-seccion="Detalles de la cuenta">
    <div v-if="usuario" class="cuenta-detalles-form">
      <p v-if="esDemo" class="cuenta-detalles-nota">Esta es una cuenta de demostración: sus datos no se pueden modificar.</p>

      <form @submit.prevent="guardarPerfil" novalidate>
        <div class="form-group">
          <label for="detalles-nombre">Nombre</label>
          <input id="detalles-nombre" v-model.trim="perfil.name" type="text" maxlength="255" :disabled="esDemo">
        </div>
        <div class="form-group">
          <label for="detalles-email">Correo electrónico</label>
          <input id="detalles-email" v-model.trim="perfil.email" type="email" maxlength="255" :disabled="esDemo">
        </div>
        <div class="form-group">
          <label for="detalles-telefono">Teléfono <span class="texto-atenuado">(opcional)</span></label>
          <input id="detalles-telefono" v-model.trim="perfil.telefono" type="tel" maxlength="30" :disabled="esDemo">
        </div>
        <p v-if="errorPerfil" class="cuenta-msg cuenta-msg-error" role="alert">{{ errorPerfil }}</p>
        <p v-if="okPerfil" class="cuenta-msg cuenta-msg-ok" role="status">{{ okPerfil }}</p>
        <button type="submit" class="btn-primary" :disabled="esDemo || guardandoPerfil || !perfilCambio">
          {{ guardandoPerfil ? 'Guardando...' : 'Guardar cambios' }}
        </button>
      </form>

      <h2 class="cuenta-subtitulo">Cambiar contraseña</h2>
      <form @submit.prevent="guardarClave" novalidate>
        <div class="form-group password-field">
          <label for="clave-actual">Contraseña actual</label>
          <input id="clave-actual" v-model="clave.actual" :type="mostrarClave.actual ? 'text' : 'password'" autocomplete="current-password" :disabled="esDemo">
          <button class="toggle-pass" :class="{activo: mostrarClave.actual}" type="button" :aria-pressed="mostrarClave.actual" :aria-label="mostrarClave.actual ? 'Ocultar contraseña actual' : 'Mostrar contraseña actual'" :title="mostrarClave.actual ? 'Ocultar contraseña actual' : 'Mostrar contraseña actual'" @click="mostrarClave.actual = !mostrarClave.actual">👁️</button>
        </div>
        <div class="form-group password-field">
          <label for="clave-nueva">Contraseña nueva</label>
          <input id="clave-nueva" v-model="clave.nueva" :type="mostrarClave.nueva ? 'text' : 'password'" autocomplete="new-password" :disabled="esDemo">
          <button class="toggle-pass" :class="{activo: mostrarClave.nueva}" type="button" :aria-pressed="mostrarClave.nueva" :aria-label="mostrarClave.nueva ? 'Ocultar contraseña nueva' : 'Mostrar contraseña nueva'" :title="mostrarClave.nueva ? 'Ocultar contraseña nueva' : 'Mostrar contraseña nueva'" @click="mostrarClave.nueva = !mostrarClave.nueva">👁️</button>
        </div>
        <div class="form-group password-field">
          <label for="clave-confirmacion">Repetir contraseña nueva</label>
          <input id="clave-confirmacion" v-model="clave.confirmacion" :type="mostrarClave.confirmacion ? 'text' : 'password'" autocomplete="new-password" :disabled="esDemo">
          <button class="toggle-pass" :class="{activo: mostrarClave.confirmacion}" type="button" :aria-pressed="mostrarClave.confirmacion" :aria-label="mostrarClave.confirmacion ? 'Ocultar confirmación de contraseña' : 'Mostrar confirmación de contraseña'" :title="mostrarClave.confirmacion ? 'Ocultar confirmación de contraseña' : 'Mostrar confirmación de contraseña'" @click="mostrarClave.confirmacion = !mostrarClave.confirmacion">👁️</button>
        </div>
        <p v-if="errorClave" class="cuenta-msg cuenta-msg-error" role="alert">{{ errorClave }}</p>
        <p v-if="okClave" class="cuenta-msg cuenta-msg-ok" role="status">{{ okClave }}</p>
        <button type="submit" class="btn-primary" :disabled="esDemo || guardandoClave">
          {{ guardandoClave ? 'Guardando...' : 'Cambiar contraseña' }}
        </button>
      </form>
    </div>
    <p v-else aria-live="polite">Cargando datos de tu cuenta…</p>
  </cuenta-shell>
</template>

<script>
import CuentaShell from '../../components/CuentaShell.vue';
import api from '../../Api/api.js';
import { esSesionDemo, obtenerUsuarioDemo } from '../../Api/demoAuth.js';
import { actualizarPerfil, cambiarClave, obtenerUsuarioGuardado } from '../../Api/cuenta.js';

export default {
  name: 'DetallesCuentaView',
  components: { CuentaShell },
  data() {
    return {
      usuario: null,
      esDemo: false,
      perfil: { name: '', email: '', telefono: '' },
      guardandoPerfil: false,
      errorPerfil: '',
      okPerfil: '',
      clave: { actual: '', nueva: '', confirmacion: '' },
      mostrarClave: { actual: false, nueva: false, confirmacion: false },
      guardandoClave: false,
      errorClave: '',
      okClave: ''
    };
  },
  computed: {
    perfilCambio() {
      if (!this.usuario) return false;
      return this.perfil.name !== (this.usuario.name || '')
        || this.perfil.email !== (this.usuario.email || '')
        || this.perfil.telefono !== (this.usuario.telefono || '');
    }
  },
  async mounted() {
    if (esSesionDemo()) {
      this.esDemo = true;
      this.cargarUsuario(obtenerUsuarioDemo());
      return;
    }
    try {
      const response = await api.get('/user');
      this.cargarUsuario(response.data?.data || response.data);
    } catch (error) {
      // Solo si el servidor dice que la sesión no vale. Si es un problema de
      // red, mostramos los datos que quedaron guardados al iniciar sesión.
      const guardado = obtenerUsuarioGuardado();
      if (error.response?.status === 401 || !guardado) {
        this.$router.push({ name: 'login' });
        return;
      }
      this.cargarUsuario(guardado);
    }
  },
  methods: {
    cargarUsuario(usuario) {
      this.usuario = usuario;
      this.perfil = { name: usuario.name || '', email: usuario.email || '', telefono: usuario.telefono || '' };
    },
    async guardarPerfil() {
      this.errorPerfil = '';
      this.okPerfil = '';
      if (!this.perfil.name) { this.errorPerfil = 'El nombre no puede quedar vacío.'; return; }
      if (!/^\S+@\S+\.\S+$/.test(this.perfil.email)) { this.errorPerfil = 'Ingresá un correo electrónico válido.'; return; }

      this.guardandoPerfil = true;
      try {
        const actualizado = await actualizarPerfil(this.perfil);
        this.cargarUsuario({ ...this.usuario, ...this.perfil, ...(actualizado || {}) });
        this.okPerfil = 'Tus datos se guardaron correctamente.';
      } catch (error) {
        this.errorPerfil = error.message;
      } finally {
        this.guardandoPerfil = false;
      }
    },
    async guardarClave() {
      this.errorClave = '';
      this.okClave = '';
      if (!this.clave.actual || !this.clave.nueva || !this.clave.confirmacion) { this.errorClave = 'Completá los tres campos.'; return; }
      if (this.clave.nueva.length < 6) { this.errorClave = 'La contraseña nueva debe tener al menos 6 caracteres.'; return; }
      if (this.clave.nueva !== this.clave.confirmacion) { this.errorClave = 'Las contraseñas nuevas no coinciden.'; return; }

      this.guardandoClave = true;
      try {
        await cambiarClave(this.clave);
        this.clave = { actual: '', nueva: '', confirmacion: '' };
        this.mostrarClave = { actual: false, nueva: false, confirmacion: false };
        this.okClave = 'Tu contraseña se cambió correctamente.';
      } catch (error) {
        this.errorClave = error.message;
      } finally {
        this.guardandoClave = false;
      }
    }
  }
};
</script>
