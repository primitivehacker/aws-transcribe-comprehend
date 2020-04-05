<template>
<div class="container">
    <div class="form-wrapper">
        <form @submit.prevent="sendFile" enctype="multipart/form-data">
            <div class="form">
                <div class="upload">
                    <div class="file is-primary">
                        <label class="file-label">
                            <input 
                                type="file" 
                                ref="file"
                                @change="selectFile"
                                class="file-input"
                            />
                            <span class="file-cta">
                                <span class="file-icon">
                                    <i class="fas fa-upload"></i>
                                </span>
                                <span class="file-label">
                                    Upload Audio
                                </span>
                            </span>
                            <span v-if="file" class="file-name">{{file.name}}</span>
                        </label>
                    </div>
                </div>
                <div class="send">
                    <button class="button is-info">Send</button>
                </div>
            </div>
        </form>
    </div>
    <div class="loading-and-results">
        <div class="progress-bar">
            <div v-if="fileUploading">
                <h1 class="title is-3">Uploading File</h1>
                <progress class="progress is-primary" max="100"></progress>
            </div>
            <div v-if="transcribeUploading" class="progress-bar">
                <h1 class="title is-3">Transcribing Audio</h1>
                <progress class="progress is-info" max="100">Transcribing Audio</progress>
            </div>
            <div v-if="comprehendUploading">
                <h1 class="title is-3">Comprehending Text</h1>
                <progress class="progress is-success" max="100">Comprehending Text</progress>
            </div>
        </div>
        <div class="results content">
            <div class="phrases" v-if="phrases.length !== 0">
                <h1 class="title is-3">Key Phrases</h1>
                <ol type="1">
                    <li v-for="phrase in phrases" :key="phrase.id">{{phrase.Text}}</li>
                </ol>
            </div>
            <div class="sentiment" v-if="sentiment">
                <h1 class="title is-3">Sentiment</h1>
                <h1 class="subtitle is-5">{{sentiment}}</h1>
            </div>
        </div>
    </div>
</div>
</template>

/////////////////////////////////////////////////////////////////////
////////////************** SCRIPT ***********////////////////////////
////////////////////////////////////////////////////////////////////

<script>
import AWS from 'aws-sdk'
import TranscribeService from "aws-sdk/clients/transcribeservice";
import Comprehend from "aws-sdk/clients/comprehend";

const uploadBucketName = "audioinputfortranscribe";
const bucketRegion = "us-east-1";
const IdentityPoolId = "us-east-1:26bbe515-3c70-4d7c-81bd-17b482593839";
const jobNameId = Math.floor(Math.random() * 10000);

AWS.config.update({
    region: bucketRegion,
});

AWS.config.credentials = new AWS.CognitoIdentityCredentials({
    IdentityPoolId: IdentityPoolId,
    RoleArn: "arn:aws:iam::375884232115:role/Cognito_AudioBucketUnauth_Role"
})

export default {
    name: "awsML",
    data() {
        return {
            file: "",
            fileUploading: false,
            transcribeUploading: false,
            comprehendUploading: false,
            phrases: [],
            sentiment: ""
        }
    },
    methods: {
        selectFile() {
            const file = this.$refs.file.files[0];
            this.file = file
        },
        async sendFile() {
            const fileName = this.file.name;
            const albumAudioKey = encodeURIComponent("audioUploadKey") + "-";
            const audioKey = albumAudioKey + fileName;

            const upload = new AWS.S3.ManagedUpload({
                params: {
                    Bucket: uploadBucketName,
                    Key: audioKey,
                    Body: this.file,
                    ACL: "public-read"
                }
            });

            const startTranscriptionJob = () => {
                this.startTranscriptionJob()
            }

            const promise = upload.promise();

            this.fileUploading = true

            console.log("UPLOAD PROGRESS", upload)

            await promise.then(
                function() {
                    startTranscriptionJob()
                },
                function(err) {
                    return alert("There was an error uploading your audio: ", err.message);
                }
            );

            this.fileUploading = false
            this.transcribeUploading = true
        },

        ////////////////////////////////////////////////////////////////////
        //////////**************START TRANSCRIPTION***********//////////////////////
        ////////////////////////////////////////////////////////////////////

        startTranscriptionJob() {
            const fileName = this.file.name;
            const albumAudioKey = encodeURIComponent("audioUploadKey") + "-";
            const audioKey = albumAudioKey + fileName;
            const transcribeservice = new TranscribeService({apiVersion: '2017-10-26'});

            const params = {
                LanguageCode: 'en-US',
                Media: { /* required */
                    MediaFileUri: `s3://audioinputfortranscribe/${audioKey}`
                },
                TranscriptionJobName: `audioJob-${jobNameId}`, /* required */
                JobExecutionSettings: {
                },
                MediaFormat: 'mp3',
                OutputBucketName: 'audiooutputfromtranscribe',
                Settings: {
                    ChannelIdentification: true,
                    ShowAlternatives: false, ShowSpeakerLabels: false,
                }
            };
            const transcribePromise = (params) => {
                return new Promise((resolve, reject) => {
                    transcribeservice.startTranscriptionJob(params, function (err, data) {
                        err ? reject(err) : resolve(data)
                    });
                })
            }
            transcribePromise(params)
                .then(data => {
                    this.startPing(data)
                })
                .catch(err => console.log(`failure: ${err}`))
        },

        ////////////////////////////////////////////////////////////////////
        //////////**************START PING***********//////////////////////
        ////////////////////////////////////////////////////////////////////

        startPing(data) {
                setInterval(this.getTranscriptionJob(data), 200000)
        },

        ////////////////////////////////////////////////////////////////////
        //////////**************GET TRANSCRIPTION***********//////////////////////
        ////////////////////////////////////////////////////////////////////

        getTranscriptionJob(data) {
            console.log("GET TRANSCRIPTION STARTED")
            console.log(data)
            const transcribeservice = new TranscribeService({apiVersion: '2017-10-26'});
            const jobName = data.TranscriptionJob.TranscriptionJobName;
            var params = {
                TranscriptionJobName: jobName 
            };

            const transcribePromise = (params) => {
                return new Promise((resolve, reject) => {
                    transcribeservice.getTranscriptionJob(params, function(err, data) {
                        err ? reject(err) : resolve(data)
                    });
                })
            }
            transcribePromise(params)
                .then(data => {
                    console.log(data.TranscriptionJob.TranscriptionJobStatus)
                    if (data.TranscriptionJob.TranscriptionJobStatus === "COMPLETED") {
                        this.startComprehend(data);
                        this.detectKeyPhrases(data);
                        this.detectSentiment(data);
                        this.transcribeUploading = false
                    } else {
                        this.startPing(data);
                    }
                })
                .catch(err => console.log(`failure: ${err}`))
        },
        ////////////////////////////////////////////////////////////////////
        //////////**************COMPREHEND***********//////////////////////
        ////////////////////////////////////////////////////////////////////
        startComprehend(data) {
            const comprehend = new Comprehend({apiVersion: '2017-11-27'});
            const jobName = data.TranscriptionJob.TranscriptionJobName;
            const params = {
                DataAccessRoleArn: 'arn:aws:iam::375884232115:role/ComprehendDataAccess', /* required */
                InputDataConfig: { /* required */
                    S3Uri: `s3://audiooutputfromtranscribe/audioJob-${jobNameId}.json`, /* required */
                },
                LanguageCode: 'en', /* required */
                OutputDataConfig: { /* required */
                    S3Uri: 's3://outputfromcomprehend/*', /* required */
                },
                JobName: jobName
            };
            comprehend.startKeyPhrasesDetectionJob(params, function(err, data) {
                console.log("STARTED PHRASE DETEC")
                if (err) console.log(err, err.stack); // an error occurred
                else {
                    console.log(data); // successful response
                    setLoadingToTrue()
                }
            });
            const setLoadingToTrue= () => {
                this.comprehendUploading = true 
            }
        },

        ////////////////////////////////////////////////////////////////////
        //////////**************KEY PHRASES***********//////////////////////
        ////////////////////////////////////////////////////////////////////
        detectKeyPhrases(data) {
            console.log("STARTED KEY PHRASES", data)
            const comprehend = new Comprehend({apiVersion: '2017-11-27'});
            const uri = data.TranscriptionJob.Transcript.TranscriptFileUri;
            fetch(uri)
                .then(res => res.json())
                .then((data) => {
                console.log('Output: ', data);
                callDetectPhrases(data.results.transcripts);
            }).catch(err => console.error(err));
            const callDetectPhrases = (data) => {
                const stringifiedUri = JSON.stringify(data)
                const trimmed = stringifiedUri.substring(0, 4900)
                const params = {
                        LanguageCode: 'en',
                        Text: trimmed 
                };
                comprehend.detectKeyPhrases(params, function(err, data) {
                    if (err) console.log(err, err.stack); // an error occurred
                    else {
                        console.log("TRANSCRIPT DATA", data);
                        stopProgressBar()
                        displayData(data)
                    }
                });
            }
            const stopProgressBar = () => {
                this.comprehendUploading = false
            }
            const displayData = (data) => {
                data.KeyPhrases.shift()
                console.log(data.KeyPhrases)
                this.phrases = data.KeyPhrases
            }
        },
        ////////////////////////////////////////////////////////////////////
        //////////**************DETECT SENTIMENT***********//////////////////////
        ////////////////////////////////////////////////////////////////////
        detectSentiment(data) {
            console.log("STARTED SENTIMENT", data)
            const comprehend = new Comprehend({apiVersion: '2017-11-27'});
            const uri = data.TranscriptionJob.Transcript.TranscriptFileUri;
            fetch(uri)
                .then(res => res.json())
                .then((data) => {
                console.log('Sentiment Output: ', data);
                callDetectSentiment(data.results.transcripts);
            }).catch(err => console.error(err));
            const callDetectSentiment = (data) => {
                const stringifiedUri = JSON.stringify(data)
                const trimmed = stringifiedUri.substring(0, 4900)
                const params = {
                        LanguageCode: 'en',
                        Text: trimmed 
                };
                comprehend.detectSentiment(params, function(err, data) {
                    if (err) console.log(err, err.stack); // an error occurred
                    else {
                        console.log("SENTIMENT DATA", data);
                        stopProgressBar()
                        displayData(data)
                    }
                });
            }
            const stopProgressBar = () => {
                this.comprehendUploading = false
            }
            const displayData = (data) => {
                console.log(data.Sentiment)
                this.sentiment = data.Sentiment
            }
        }
    }
}
</script>

<style>
.container{
    display: grid;
    grid-template-columns: auto auto auto auto;
    grid-column-gap: 50px;
    grid-row-gap: 100px;
}
.form{
    display: grid;
    grid-template-columns: auto auto;
    grid-row-gap: 10px;
}
.upload{
    grid-column-start: 1;
    grid-column-end: 5;
}
.send{
    grid-column-start: 1;
    grid-column-end: 5;
}
.loading-and-results{
    grid-column-start: 1;
    grid-column-end: 5;
    
}
.progress-bar{
    grid-column-start: 1;
    grid-column-end: 5;
    align-items: center;
}
.results{
    display: grid;
    grid-template-columns: auto auto auto auto;
}
.phrases {
    grid-column-start: 1;
    grid-column-end: 3;
}
.sentiment{
    grid-column-start: 3;
    grid-column-end: 5;
}
</style>