<style>
</style>

<template>
    <div class="ui main text container" style="margin-top: 80px;">
        <div class="ui container center aligned">
            <div class="ui basic segment huge">
                <h1>
                    <i class="circular icon add user"></i>
                    Create Bot User
                </h1>
                This bot will send and receive messages autonomously <br />
                as a <strong>new</strong> Forsta user you create.
                <br /><br />
            </div>
            <div class="ui centered grid">
                <div class="ui nine wide column basic segment left aligned t0 b1">
                    <form v-on:submit.prevent="completeAuth" class="ui huge form enter-code" :class="{loading: loading}">
                        <div class="field">
                            <label>Forsta {{label}}</label>
                            <div class="ui left icon input">
                                <input v-focus.lazy="true" :type="inputType" name="secret" :placeholder="placeholder" autocomplete="off" v-model='secret'>
                                <i class="lock icon"></i>
                            </div>
                        </div>
                        <div class="field" v-if="type === 'totp'">
                            <label>Authentication Code</label>
                            <div class="ui left icon input">
                                <input type="number" name="otp" placeholder="authentication code" autocomplete="off" v-model='otp'>
                                <i class="clock icon"></i>
                            </div>
                        </div>
                        <button class="ui large primary submit button right floated" type="submit">Submit</button>
                        <router-link :to="{name: 'onboardTag'}" class="ui large button secret-cancel">Cancel</router-link>
                    </form>
                    <sui-message size="small" negative v-if="error" :content="error" />
                </div>
            </div>
        </div>
    </div>
</template>

<script>
util = require('../util');
shared = require('../globalState');
focus = require('vue-focus');

module.exports = {
    data: () => ({
        secret: '',
        otp: '',
        type: '',
        loading: false,
        error: '',
        global: shared.state
    }),
    computed: {
        placeholder: function () {
            return this.type === 'sms' ? '000000' : 'password';
        },
        label: function () {
            return this.type === 'sms' ? 'SMS Code' : 'Password';
        },
        inputType: function () {
            return this.type === 'sms' ? 'text' : 'password';
        }
    },
    mounted: function() {
        this.type = this.$route.params.type;
        util.fetch.call(this, '/api/auth/status/v1')
        .then(result => { 
            this.global.onboardStatus = result.theJson.status;
            if (this.global.onboardStatus === 'complete') {
                this.$router.push({name: 'loginTag'});
            }
        });
    },
    methods: {
        validate: function () {
            if (!this.secret) return this.type === 'sms' ? 'SMS code is required.' : 'Password is required.';
            if (this.type === 'sms') return this.secret.match(/^\d{6}$/) ? '' : 'Please enter the 6-digit code you were sent.';
            if (this.type === 'totp' && !this.otp) return 'Authentication code is required.'
        },
        completeAuth () {
            this.error = this.validate();
            if (this.error) return;

            var tag = this.$route.params.tag;
            var value = this.secret;
            var type = this.type;
            var otp = this.otp;
            this.loading = true;
            util.fetch.call(this, '/api/auth/atlasauth/authenticate/v1/' + tag, { method: 'post', body: { value, type, otp }})
            .then(result => {
                this.loading = false;
                if (result.ok) {
                    const { token } = result.theJson;
                    this.global.apiToken = token;
                    this.$router.push({ name: 'onboardComplete' });
                    return false;
                } else {
                    this.error = util.mergeErrors(result.theJson) || 'Internal error, please try again.';
                    return false;
                }
            })
            .catch(err => {
                this.error = util.mergeErrors(result.theJson) || 'Internal error, please try again.';
                this.loading = false;
            });
            return false;
        }
    },
    directives: {
        focus: focus.focus
    }
}
</script>