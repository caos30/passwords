<template>
    <div id="app-content">
        <form id="authorize-window" @submit="submitLogin($event)" :class="getClasses">
            <translate tag="div"
                       icon="user"
                       class="login-message"
                       say="You are logging in as {user}"
                       :variables="loginVars"
                       v-if="impersonating"/>
            <div class="login-container">
                <translate icon="repeat"
                           title="Request token again"
                           :iconClass="retryClass"
                           @click="requestToken()"
                           v-if="retryVisible"/>
                <div>
                    <field type="password" placeholder="Password" v-model="password" :disabled="loggingIn" required v-if="hasPassword"/>
                    <select v-model="providerId" :disabled="loggingIn" v-if="hasToken && providers.length > 0">
                        <option v-for="(option, id) in providers" :value="id" :title="option.description">
                            {{ option.label }}
                        </option>
                    </select>
                    <field placeholder="Token"
                           :title="provider.description"
                           v-model="token"
                           required
                           :disabled="loggingIn"
                           v-if="showTokenField"/>
                </div>
            </div>
            <div class="login-confirm" v-if="hasToken || hasPassword">
                <translate tag="input" type="submit" localized-value="Login" :class="{'no-icon':loggingIn}"/>
                <div class="login-icon fa fa-circle-o-notch fa-spin" v-if="loggingIn">&nbsp;</div>
                <translate class="login-error" :say="errorMessage" tag="div" v-if="hasError"/>
            </div>
        </form>
    </div>
</template>

<script>
    import Field from '@vc/Field';
    import API from '@js/Helper/api';
    import Translate from '@vc/Translate';
    import Messages from '@js/Classes/Messages';
    import SetupManager from '@js/Manager/SetupManager';

    export default {
        components: {Field, Translate},
        data() {
            let impersonateEl = document.querySelector('meta[name=pw-impersonate]'),
                impersonating = false;

            if(impersonateEl) impersonating = impersonateEl.getAttribute('content') === 'true';

            return {
                password    : '',
                token       : '',
                salts       : [],
                providerId  : -1,
                provider    : null,
                providers   : [],
                hasPassword : false,
                hasToken    : false,
                hasError    : false,
                pwAlgorithm : '',
                errorMessage: '',
                loggingIn   : false,
                retryClass  : '',
                impersonating
            };
        },

        created() {
            document.body.classList.remove('pw-auth-skipped', 'pw-auth-passed');
            document.body.classList.add('pw-auth-visible');
            API.requestSession()
               .then((d) => {
                   if(d.hasOwnProperty('challenge')) {
                       this.hasPassword = true;
                       this.salts = d.challenge.salts;
                   }
                   if(d.hasOwnProperty('token')) {
                       this.hasToken = true;
                       this.provider = null;
                       this.providerId = -1;
                       this.providers = [];

                       for(let i = 0; i < d.token.length; i++) {
                           if(d.token[i].id === 'twofactor_nextcloud_notification' && d.token.length !== 1 && APP_ENVIRONMENT !== 'development') {
                               continue;
                           }

                           this.providers.push(d.token[i]);
                           if(this.provider === null && !d.token[i].request) {
                               this.providerId = this.providers.length - 1;
                               this.provider = d.token[i];
                           }
                       }

                       if(this.provider === null) {
                           this.providerId = 0;
                           this.provider = this.providers[0];
                       }
                   }
                   if(!this.hasPassword && !this.hasToken) {
                       API.openSession([])
                          .then(() => { this.goToTarget(); })
                          .catch((d) => { this.loginError(d); });
                   }
               });
        },

        computed: {
            retryVisible() {
                return this.provider !== null && this.provider.request;
            },
            loginVars() {
                return {
                    user: document.head.getAttribute('data-user-displayname')
                };
            },
            getClasses() {
                return '' + (this.hasPassword ? ' has-password':'') + (this.hasToken ? ' has-token':'');
            },
            showTokenField() {
                return this.hasToken && this.provider.type === 'user-token';
            }
        },

        methods: {
            submitLogin(e) {
                e.preventDefault();
                if(this.loggingIn) return;
                this.loggingIn = true;
                let data = {};

                if(this.hasPassword) {
                    data.password = this.password;
                    data.salts = this.salts;
                }
                if(this.hasToken) {
                    data.token = {};
                    data.token[this.provider.id] = this.token;
                }
                this.hasError = false;

                this.$nextTick(() => {
                    API.openSession(data)
                       .then(() => { this.goToTarget(); })
                       .catch((d) => { this.loginError(d); });
                });
            },
            loginError(e) {
                this.password = '';
                if(this.hasToken && this.provider.type !== 'user-token') this.token = '';
                this.hasError = true;

                if(e.response && e.response.status === 403) {
                    this.errorMessage = 'Password invalid. Session revoked for too many failed login attempts.';

                    setTimeout('location.reload()', 2500);
                    return;
                } else if(e.message) {
                    this.errorMessage = e.message;
                } else {
                    this.errorMessage = 'Unknown Error';
                }
                this.loggingIn = false;
            },
            goToTarget() {
                let target = this.$route.params.target,
                    route  = {path: '/'};

                if(target) route = JSON.parse(atob(target));
                document.body.classList.add('pw-auth-passed');

                setTimeout(() => {
                    this.$router.push(route);
                }, 250);

                setTimeout(() => {
                    document.body.classList.remove('pw-auth-visible');
                    SetupManager.runAutomatically();
                }, 500);
            },
            requestToken() {
                this.retryClass = 'fa-spin';
                API.requestToken(this.provider.id)
                   .then((d) => {
                       if(this.provider.type === 'request-token') this.token = d.data.token;
                       setTimeout(() => {this.retryClass = '';}, 1500);
                   })
                   .catch(() => {
                       Messages.alert(
                           'You may have requested too many tokens. Please try again later.',
                           'Token request failed'
                       );
                       this.retryClass = '';
                   });
            }
        },

        watch: {
            providerId(value) {
                this.provider = this.providers[value];
                this.token = '';
                if(this.provider.request) {
                    this.requestToken();
                }
            }
        }
    };
</script>

<style lang="scss">
body#body-user {
    &:not(.pw-auth-passed):not(.pw-auth-skipped) {
        background        : var(--color-primary) var(--pw-image-login-background);
        background-size   : cover;
        background-repeat : no-repeat;

        #header {
            background : rgba(0, 0, 0, 0) none !important;
        }
    }

    #appmenu li a::before {
        display : none;
    }

    #content-wrapper {
        padding-top : 0;
    }

    #app-navigation {
        transform  : translateX(-100%);
        transition : transform ease-in-out 300ms;
    }
}

#authorize-window {
    margin     : 0 auto;
    width      : 300px;
    text-align : center;
    position   : relative;

    .fa-repeat {
        position  : absolute;
        font-size : 1.5rem;
        color     : var(--color-primary-text);
        padding   : 0.5rem;
        cursor    : pointer;
        right     : -2.5rem;
        width     : 2.5rem;
    }

    &.has-password {
        .fa-repeat {
            top : 3rem;
        }
    }

    .login-container {
        position : relative;
    }

    input {
        width     : 100%;
        padding   : 0.75rem;
        border    : none;
        font-size : 1rem;
        height    : auto;

        &[type=text],
        &[type=password] {
            margin        : 0;
            box-shadow    : 0 1px 0 transparentize($color-black, 0.9) inset !important;
            border-radius : 0;

            &:first-child {
                box-shadow              : none;
                border-top-left-radius  : 0.25rem;
                border-top-right-radius : 0.25rem;
            }

            &:last-child {
                border-bottom-left-radius  : 0.25rem;
                border-bottom-right-radius : 0.25rem;
            }
        }
    }

    select {
        width            : 100%;
        border           : none;
        font-size        : 1rem;
        box-shadow       : 0 1px 0 transparentize($color-black, 0.9) inset !important;
        border-radius    : 0;
        background-color : var(--color-main-background);
        padding          : 0.75rem 0.75rem 0.75rem 0.5rem;
        margin           : 0;
        height           : 2.75rem;

        &:first-child {
            box-shadow              : none;
            border-top-left-radius  : 0.25rem;
            border-top-right-radius : 0.25rem;
        }

        &:last-child {
            border-bottom-left-radius  : 0.25rem;
            border-bottom-right-radius : 0.25rem;
        }
    }

    .login-message {
        padding          : 0.75rem 0.25rem 0.75rem 1.75rem;
        margin-bottom    : 0.75rem;
        background-color : var(--color-box-shadow);
        color            : var(--color-primary-text);
        font-weight      : bold;
        font-size        : 1rem;
        border-radius    : var(--border-radius);
        position         : relative;

        i {
            position    : absolute;
            left        : 0.5rem;
            top         : 0;
            bottom      : 0;
            display     : flex;
            align-items : center;
        }
    }

    .login-confirm {
        position : relative;

        .login-icon {
            position    : absolute;
            width       : 1rem;
            font-size   : 1rem;
            line-height : 1rem;
            right       : 16px;
            top         : 2rem;
            color       : var(--color-primary-text);
        }

        input[type=submit] {
            margin-top            : 1rem;
            border                : 1px solid var(--color-primary-text);
            color                 : var(--color-primary-text);
            background            : var(--color-primary-element) var(--icon-confirm-fff) no-repeat;
            background-position-y : 50%;
            background-position-x : calc(100% - 16px);
            transition            : background-position-x 0.25s ease-in-out;

            &.no-icon {
                background-image : none;
            }

            &:hover {
                background-position-x : calc(100% - 8px);
            }
        }
    }

    .login-error {
        color            : var(--color-error);
        margin-top       : 1rem;
        background-color : var(--color-box-shadow);
        border-radius    : var(--border-radius);
        padding          : 0.5rem;
        border           : 1px solid var(--color-error);
        position         : absolute;
        width            : 300px;
        animation        : shake 0.1s ease-in-out 0s 5;

        @keyframes shake {
            0% {
                margin-left : -5px;
            }
            33% {
                margin-left : 0;
            }
            66% {
                margin-left : 5px;
            }
            99% {
                margin-left : 0;
            }
        }
    }
}

body#body-user {
    &.pw-auth-visible {
        #app-content {
            position          : fixed;
            top               : 0;
            left              : 0;
            bottom            : 0;
            right             : 0;
            margin-left       : 0 !important;
            display           : flex;
            align-items       : center;
            background        : var(--color-primary) var(--pw-image-login-background);
            background-size   : cover;
            background-repeat : no-repeat;
            opacity           : 1;
            transition        : opacity ease-in-out 0.25s, margin-left ease-in-out 0.25s 0.25s;

            > * {
                transition : opacity ease-in-out 0.25s;
            }
        }

        #app-navigation {
            z-index : 1001;
        }

        &.pw-auth-passed,
        &.pw-auth-skipped {
            #app-content {
                background-image : none;
                background-color : var(--color-main-background);

                > * {
                    opacity : 0;
                }
            }
        }
    }

    &.pw-auth-passed {
        #header {
            transition       : background-color ease-in-out 0.25s 0.25s;
            background-color : var(--color-primary);
        }

        @media (min-width : $width-small) {
            #app-navigation {
                transform : translateX(0);
            }
        }
    }

    &.pw-auth-skipped {
        #app-content {
            > * {
                transition : opacity ease-in-out 0.25s;
            }
        }

        @media (min-width : $width-small) {
            #app-navigation {
                transform : translateX(0);
            }
        }
    }
}
</style>