<script>
import gql from 'graphql-tag';
import { useReCaptcha } from 'vue-recaptcha-v3';
import { provideApolloClient } from '@vue/apollo-composable';
import { ApolloClient, InMemoryCache } from '@apollo/client/core';

const defaultFormData = {
    workPhone: '',
    subject: '',
    message: '',
    lastName: '',
    howMuchDoYouEnjoyEatingPie: '',
    howDidYouHearAboutThisJobPosting: [],
    homePhone: '',
    firstName: '',
    email: '',
    department: '',
    companyName: '',
    cellPhone: '',
    appointmentDate: '',
    acceptTerms: '',
};

const defaultFormProperties = {
    csrf: {
        name: '',
        token: '',
    },
    honeypot: {
        name: '',
        value: '',
    },
    reCaptcha: {
        enabled: false,
        handle: '',
        name: '',
    },
    loadingText: '',
    successMessage: '',
    errorMessage: '',
};

const client = new ApolloClient({
    uri: 'https://demo.solspace.net/craft/graphql/api',
    headers: {
        'Cache-Control': 'no-cache',
        'Accept': 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
        'HTTP_X_REQUESTED_WITH': 'XMLHttpRequest',
        'X-Craft-Solspace-Freeform-Mode': 'Headless',
    },
    cache: new InMemoryCache(),
});

provideApolloClient(client);

async function getFormProperties(formId) {
    // See https://docs.solspace.com/craft/freeform/v4/developer/graphql/#how-to-render-a-form
    const response = await fetch(`https://demo.solspace.net/craft/freeform/form/properties/${formId}`, { headers: { 'Accept': 'application/json' }});

    if (!response.ok) {
        throw new Error('Failed to fetch Craft Freeform Form properties');
    }

    return response.json();
}

async function saveQuoteSubmission(params) {
    const { reCaptchaValue, formData, formProperties } = params;
    const { csrf, honeypot, reCaptcha } = formProperties;

    return await client.mutate({
        mutation: gql`
            mutation SaveQuoteSubmission(
                $honeypot: FreeformHoneypotInputType,
                $reCaptcha: FreeformReCaptchaInputType,
                $csrfToken: FreeformCsrfTokenInputType,
                $workPhone: String,
                $subject: String,
                $message: String,
                $lastName: String,
                $howMuchDoYouEnjoyEatingPie: String,
                $howDidYouHearAboutThisJobPosting: [String],
                $homePhone: String,
                $firstName: String,
                $email: String,
                $department: String,
                $companyName: String,
                $cellPhone: String,
                $appointmentDate: DateTime,
                $acceptTerms: String
            ) {
                save_quote_Submission(
                    honeypot: $honeypot
                    reCaptcha: $reCaptcha
                    csrfToken: $csrfToken
                    workPhone: $workPhone
                    subject: $subject
                    message: $message
                    lastName: $lastName
                    howMuchDoYouEnjoyEatingPie: $howMuchDoYouEnjoyEatingPie
                    howDidYouHearAboutThisJobPosting: $howDidYouHearAboutThisJobPosting
                    homePhone: $homePhone
                    firstName: $firstName
                    email: $email
                    department: $department
                    companyName: $companyName
                    cellPhone: $cellPhone
                    appointmentDate: $appointmentDate
                    acceptTerms: $acceptTerms
                ) {
                    submissionId
                    success
                }
        }
        `,
        variables: {
            honeypot: {
                name: honeypot.name,
                value: honeypot.value,
            },
            csrfToken: {
                name: csrf.name,
                value: csrf.token,
            },
            reCaptcha: {
                name: reCaptcha.name,
                value: reCaptchaValue,
            },
            firstName: formData.firstName,
            lastName: formData.lastName,
            companyName: formData.companyName,
            email: formData.email,
            cellPhone: formData.cellPhone,
            homePhone: formData.homePhone,
            workPhone: formData.workPhone,
            subject: formData.subject,
            appointmentDate: formData.appointmentDate,
            department: formData.department,
            howMuchDoYouEnjoyEatingPie: formData.howMuchDoYouEnjoyEatingPie,
            message: formData.message,
            howDidYouHearAboutThisJobPosting: formData.howDidYouHearAboutThisJobPosting,
            acceptTerms: formData.acceptTerms,
        },
    });
}

export default {
    name: 'QuoteForm',
    data: () => ({
        submitButton: null,
        errorMessage: null,
        successMessage: null,
        reCaptchaValue: null,
        formData: defaultFormData,
        formProperties: defaultFormProperties,
    }),
    setup() {
        const { executeRecaptcha, recaptchaLoaded } = useReCaptcha();

        const getReCaptcha = async () => {
            await recaptchaLoaded();

            return await executeRecaptcha('setup');
        };

        return {
            getReCaptcha,
        };
    },
    created() {
        const formId = 4;

        getFormProperties(formId).then(formProperties => {
            this.formProperties = formProperties;
        });

        this.getReCaptcha().then(reCaptchaValue => {
            this.reCaptchaValue = reCaptchaValue;
        });
    },
    mounted() {
        this.errorMessage = document.querySelector('#errorMessage');
        this.successMessage = document.querySelector('#successMessage');
        this.submitButton = document.querySelector('button[type="submit"]');

        this.hideError();
        this.hideSuccess();
    },
    methods: {
        handleHowDidYouHearAboutThisJobPosting(event) {
            let howDidYouHearAboutThisJobPosting = [...this.formData.howDidYouHearAboutThisJobPosting];

            if (event.target.checked) {
                howDidYouHearAboutThisJobPosting.push(event.target.value);
            } else {
                howDidYouHearAboutThisJobPosting = howDidYouHearAboutThisJobPosting.filter((value) => value !== event.target.value);
            }

            this.formData = {
                ...this.formData,
                howDidYouHearAboutThisJobPosting,
            };
        },
        startProcessing() {
            this.submitButton.style.cursor = 'not-allowed';
            this.submitButton.innerText = this.formProperties.loadingText;
        },
        stopProcessing() {
            this.submitButton.innerText = 'Submit';
            this.submitButton.style.cursor = 'pointer';
        },
        showSuccess() {
            this.successMessage.style.display = 'block';
            this.scrollToTop();
        },
        hideSuccess() {
            this.successMessage.style.display = 'none';
        },
        showError(error) {
            console.error(error);

            this.errorMessage.style.display = 'block';
            this.scrollToTop();
        },
        hideError() {
            this.errorMessage.style.display = 'none';
        },
        scrollToTop() {
            window.scrollTo({ top: 0, behavior: 'smooth' });
        },
        async handleSubmit(event) {
            this.hideError();
            this.hideSuccess();
            this.startProcessing();

            const formData = this.formData;
            const formProperties = this.formProperties;
            const reCaptchaValue = this.reCaptchaValue;

            try {
                const response = await saveQuoteSubmission({ reCaptchaValue, formData, formProperties });

                if (response && response.data && response.data['save_quote_Submission'] && response.data['save_quote_Submission'].success) {
                    this.showSuccess();
                } else if (response && response.errors) {
                    this.showError(response.errors);
                }
            } catch (error) {
                this.showError(error);
            }

            this.stopProcessing();
            event.target.reset();
        },
    },
};
</script>

<template>
    <div id="successMessage" class="bg-green-100 border border-green-400 text-green-700 px-4 py-3 rounded relative mb-4" style="display: none;">
        <p>{{ this.formProperties.successMessage }}</p>
    </div>
    <div id="errorMessage" class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded relative mb-4" style="display: none;">
        <p>{{ this.formProperties.errorMessage }}</p>
    </div>
    <form class="text-center flex flex-col items-left justify-left" @submit.prevent="handleSubmit">
        <h3 class="mb-4 text-xl font-normal text-left">Quote Form</h3>
        <div class="flex flex-col w-full space-y-3">
            <div class="form-row">
                <div class="field-wrapper">
                    <label for="firstName">First Name <span class="ml-1 text-[red]">*</span></label>
                    <input class="form-input field-input" name="firstName" type="text" id="firstName" v-model="formData.firstName" v-on:change="event => (formData = { ...formData, firstName: event.target.value })" required />
                </div>
                <div class="field-wrapper">
                    <label for="lastName">Last Name <span class="ml-1 text-[red]">*</span></label>
                    <input class="form-input field-input" name="lastName" type="text" id="lastName" v-model="formData.lastName" v-on:change="event => (formData = { ...formData, lastName: event.target.value })" required />
                </div>
            </div>
            <div class="form-row">
                <div class="field-wrapper">
                    <label for="companyName">Organization Name</label>
                    <input class="form-input field-input" name="companyName" type="text" id="companyName" v-model="formData.companyName" v-on:change="event => (formData = { ...formData, companyName: event.target.value })" />
                </div>
            </div>
            <div class="form-row">
                <div class="field-wrapper">
                    <label for="email">Email <span class="ml-1 text-[red]">*</span></label>
                    <div class="text-xs italic text-slate-500">We&apos;ll never share your email with anyone else.</div>
                    <input class="form-input field-input" name="email" type="email" id="email" v-model="formData.email" v-on:change="event => (formData = { ...formData, email: event.target.value })" required />
                </div>
            </div>
            <div class="form-row">
                <div class="field-wrapper">
                    <label for="cellPhone">Cell Phone <span class="ml-1 text-[red]">*</span></label>
                    <input class="form-input field-input" name="cellPhone" type="tel" id="cellPhone" v-model="formData.cellPhone" v-on:change="event => (formData = { ...formData, cellPhone: event.target.value })" required />
                </div>
                <div class="field-wrapper">
                    <label for="homePhone">Home Phone</label>
                    <input class="form-input field-input" name="homePhone" type="tel" id="homePhone" v-model="formData.homePhone" v-on:change="event => (formData = { ...formData, homePhone: event.target.value })" />
                </div>
                <div class="field-wrapper">
                    <label for="workPhone">Work Phone</label>
                    <input class="form-input field-input" name="workPhone" type="tel" id="workPhone" v-model="formData.workPhone" v-on:change="event => (formData = { ...formData, workPhone: event.target.value })" />
                </div>
            </div>
            <div class="form-row">
                <div class="field-wrapper">
                    <label for="subject">Subject <span class="ml-1 text-[red]">*</span></label>
                    <select class="form-select field-input" name="subject" id="subject" v-model="formData.subject" v-on:change="event => (formData = { ...formData, subject: event.target.value })" required>
                        <option value="">I need some help with...</option>
                        <option value="myHomework">My homework</option>
                        <option value="practicingMyHammerDance">Practicing my hammer dance</option>
                        <option value="findingMyBellyButton">Finding my belly button</option>
                    </select>
                </div>
                <div class="field-wrapper">
                    <label for="appointmentDate">Appointment Date</label>
                    <input class="form-input field-input" name="appointmentDate" type="text" id="appointmentDate" placeholder="YYYY/MM/DD" autoComplete="off" v-model="formData.appointmentDate" v-on:change="event => (formData = { ...formData, appointmentDate: event.target.value })"  />
                </div>
                <div class="field-wrapper">
                    <label for="department">Department <span class="ml-1 text-[red]">*</span></label>
                    <select class="form-select field-input" name="department" id="department" v-model="formData.department" v-on:change="event => (formData = { ...formData, department: event.target.value })" required>
                        <option value="">Please choose...</option>
                        <option value="sales@example.com">Sales</option>
                        <option value="service@example.com">Service</option>
                        <option value="support@example.com">Support</option>
                    </select>
                </div>
            </div>
            <div class="form-row">
                <div class="field-wrapper">
                    <label for="howMuchDoYouEnjoyEatingPie-1" class="flex flex-row">How much do you enjoy eating pie?</label>
                    <div class="flex flex-row space-x-4">
                        <label for="howMuchDoYouEnjoyEatingPie-1" class="flex flex-row items-center justify-center">
                            <input class="field-input-radio" name="howMuchDoYouEnjoyEatingPie" type="radio" id="howMuchDoYouEnjoyEatingPie-1" value="1" v-on:change="event => (formData = { ...formData, howMuchDoYouEnjoyEatingPie: event.target.value })" /> 1
                        </label>
                        <label for="howMuchDoYouEnjoyEatingPie-2" class="flex flex-row items-center justify-center">
                            <input class="field-input-radio" name="howMuchDoYouEnjoyEatingPie" type="radio" id="howMuchDoYouEnjoyEatingPie-2" value="2" v-on:change="event => (formData = { ...formData, howMuchDoYouEnjoyEatingPie: event.target.value })" /> 2
                        </label>
                        <label for="howMuchDoYouEnjoyEatingPie-3" class="flex flex-row items-center justify-center">
                            <input class="field-input-radio" name="howMuchDoYouEnjoyEatingPie" type="radio" id="howMuchDoYouEnjoyEatingPie-3" value="3" v-on:change="event => (formData = { ...formData, howMuchDoYouEnjoyEatingPie: event.target.value })" /> 3
                        </label>
                        <label for="howMuchDoYouEnjoyEatingPie-4" class="flex flex-row items-center justify-center">
                            <input class="field-input-radio" name="howMuchDoYouEnjoyEatingPie" type="radio" id="howMuchDoYouEnjoyEatingPie-4" value="4" v-on:change="event => (formData = { ...formData, howMuchDoYouEnjoyEatingPie: event.target.value })" /> 4
                        </label>
                        <label for="howMuchDoYouEnjoyEatingPie-5" class="flex flex-row items-center justify-center">
                            <input class="field-input-radio" name="howMuchDoYouEnjoyEatingPie" type="radio" id="howMuchDoYouEnjoyEatingPie-5" value="5" v-on:change="event => (formData = { ...formData, howMuchDoYouEnjoyEatingPie: event.target.value })" /> 5
                        </label>
                    </div>
                </div>
            </div>
            <div class="form-row">
                <div class="field-wrapper">
                    <label for="message">Message <span class="ml-1 text-[red]">*</span></label>
                    <textarea class="form-textarea field-input" name="message" id="message" rows={5} v-model="formData.message" v-on:change="event => (formData = { ...formData, message: event.target.value })" required></textarea>
                </div>
            </div>
            <div class="form-row">
                <div class="field-wrapper">
                    <label for="howDidYouHearAboutThisJobPosting-Newspaper">How did you hear about us?</label>
                    <label class="flex flex-row items-center justify-center">
                        <input class="field-input-checkbox" name="howDidYouHearAboutThisJobPosting[]" type="checkbox" id="howDidYouHearAboutThisJobPosting-Newspaper" value="Newspaper" v-on:change="handleHowDidYouHearAboutThisJobPosting" /> Newspaper
                    </label>
                    <label class="flex flex-row items-center justify-center">
                        <input class="field-input-checkbox" name="howDidYouHearAboutThisJobPosting[]" type="checkbox" id="howDidYouHearAboutThisJobPosting-Radio" value="Radio" v-on:change="handleHowDidYouHearAboutThisJobPosting" /> Radio
                    </label>
                    <label class="flex flex-row items-center justify-center">
                        <input class="field-input-checkbox" name="howDidYouHearAboutThisJobPosting[]" type="checkbox" id="howDidYouHearAboutThisJobPosting-CarrierPigeon" value="Carrier Pigeon" v-on:change="handleHowDidYouHearAboutThisJobPosting" /> Carrier Pigeon
                    </label>
                    <label class="flex flex-row items-center justify-center">
                        <input class="field-input-checkbox" name="howDidYouHearAboutThisJobPosting[]" type="checkbox" id="howDidYouHearAboutThisJobPosting-Other" value="Other" v-on:change="handleHowDidYouHearAboutThisJobPosting" /> Other
                    </label>
                </div>
            </div>
            <div class="form-row">
                <div class="field-wrapper">
                    <label for="acceptTerms" class="flex flex-row items-center justify-center">
                        <input class="field-input-checkbox" name="acceptTerms" type="checkbox" id="acceptTerms" value="yes" v-on:change="event => (formData = { ...formData, acceptTerms: event.target.checked ? event.target.value : '' })" required />
                        I agree to the <a href="https://solspace.com" class="mx-1 underline">terms &amp; conditions</a> required by this site. <span class="ml-1 text-[red]">*</span>
                    </label>
                </div>
            </div>
            <div class="form-row">
                <div class="flex flex-row items-left justify-left space-y-2 w-full">
                    <button class="btn-primary" type="submit">Submit</button>
                </div>
            </div>
        </div>
    </form>
</template>

<style scoped>
</style>