<!--
Copyright (c) 2018 by SAP SE or an SAP affiliate company. All rights reserved. This file is licensed under the Apache Software License, v. 2 except as noted otherwise in the LICENSE file

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
 -->

<template>
  <v-dialog v-model="visible" persistent max-width="1200" content-class="dialogContainer">
    <v-card flat>

      <v-toolbar dense class="draken-1 header">
        <v-toolbar-title>
          <v-icon x-large class="white--text">mdi-hexagon-multiple</v-icon>
          <span>Create Cluster</span>
        </v-toolbar-title>
        <v-tabs align-with-title dark slot="extension" v-model="activeTab">
          <v-tabs-slider color="yellow"></v-tabs-slider>
          <v-tab key="infra"  href="#tab-infra"  ripple>Infrastructure</v-tab>
          <v-tab key="worker" href="#tab-worker" ripple>Worker</v-tab>
          <v-tab key="addons" href="#tab-addons" ripple>Addons</v-tab>
          <v-tab key="maintenance" href="#tab-maintenance" ripple>Maintenance</v-tab>
        </v-tabs>
      </v-toolbar>
      <v-tabs-items v-model="activeTab">
        <v-tab-item key="infra" id="tab-infra">
          <v-card flat>
            <v-container fluid>
              <v-card-text>

                <v-layout row>
                  <v-flex xs3>
                    <v-text-field
                      ref="name"
                      color="cyan"
                      label="Cluster Name"
                      counter="10"
                      v-model="clusterName"
                      :error-messages="getErrorMessages('shootDefinition.metadata.name')"
                      @input="$v.shootDefinition.metadata.name.$touch()"
                      @blur="$v.shootDefinition.metadata.name.$touch()"
                      required
                      ></v-text-field>
                  </v-flex>
                </v-layout>

                <v-layout row class="mt-2">
                  <v-flex xs3>
                    <v-select
                      color="cyan"
                      label="Infrastructure"
                      :items="sortedCloudProviderKindList"
                      v-model="infrastructureKind"
                      required
                      >
                      <template slot="item" slot-scope="data">
                        <v-list-tile-avatar>
                          <infra-icon v-model="data.item"></infra-icon>
                        </v-list-tile-avatar>
                        <v-list-tile-content>
                          <v-list-tile-title>{{data.item}}</v-list-tile-title>
                        </v-list-tile-content>
                      </template>
                      <template slot="selection" slot-scope="data">
                        <v-avatar size="30px">
                          <infra-icon v-model="data.item"></infra-icon>
                        </v-avatar>
                        <span class="black--text">
                          {{data.item}}
                        </span>
                      </template>
                    </v-select>
                  </v-flex>

                  <v-flex xs1 v-show="cloudProfiles.length !== 1">
                  </v-flex>

                  <v-flex xs3 v-show="cloudProfiles.length !== 1">
                    <cloud-profile
                      ref="cloudProfile"
                      v-model="cloudProfileName"
                      :isCreateMode="true"
                      :cloudProfiles="cloudProfiles"
                      color="cyan">
                    </cloud-profile>
                  </v-flex>

                  <v-flex xs1>
                  </v-flex>

                  <v-flex xs3>
                    <v-select
                      color="cyan"
                      label="Secrets"
                      :items="infrastructureSecretsByProfileName"
                      item-value="metadata"
                      v-model="secret"
                      :error-messages="getErrorMessages('shootDefinition.spec.cloud.secretBindingRef.name')"
                      @input="$v.shootDefinition.spec.cloud.secretBindingRef.name.$touch()"
                      @blur="$v.shootDefinition.spec.cloud.secretBindingRef.name.$touch()"
                      required
                      >
                      <template slot="item" slot-scope="data">
                        {{get(data.item, 'metadata.name')}}
                        <v-icon v-if="!isOwnSecretBinding(data.item)">mdi-share</v-icon>
                      </template>
                      <template slot="selection" slot-scope="data">
                        <span class="black--text">
                          {{get(data.item, 'metadata.name')}}
                        </span>
                        <v-icon v-if="!isOwnSecretBinding(data.item)">mdi-share</v-icon>
                      </template>
                    </v-select>
                  </v-flex>
                </v-layout>

                <v-layout row>
                  <v-flex xs3>
                    <v-select
                      color="cyan"
                      label="Region"
                      :items="regions"
                      v-model="region"
                      :error-messages="getErrorMessages('shootDefinition.spec.cloud.region')"
                      @input="$v.shootDefinition.spec.cloud.region.$touch()"
                      @blur="$v.shootDefinition.spec.cloud.region.$touch()"
                      required
                      ></v-select>
                  </v-flex>
                  <v-flex xs1>
                  </v-flex>
                  <v-flex xs3 v-if="infrastructureKind !== 'azure'">
                    <v-select
                      color="cyan"
                      label="Zone"
                      :items="zones"
                      :error-messages="getErrorMessages('infrastructureData.zones')"
                      v-model="zone"
                      @input="$v.infrastructureData.zones.$touch()"
                      @blur="$v.infrastructureData.zones.$touch()"
                      required
                      ></v-select>
                  </v-flex>
                </v-layout>

                <v-layout row>
                  <v-flex xs3>
                    <v-select
                      color="cyan"
                      label="Kubernetes"
                      :items="sortedKubernetesVersions"
                      v-model="shootDefinition.spec.kubernetes.version"
                      required
                      ></v-select>
                  </v-flex>
                  <v-flex xs1>
                  </v-flex>
                  <v-flex xs3>
                    <v-select
                      color="cyan"
                      label="Purpose"
                      :items="purpose"
                      v-model="shootDefinition.metadata.annotations['garden.sapcloud.io/purpose']"
                      hint="Indicate the importance of the cluster"
                      persistent-hint
                      required
                      ></v-select>
                  </v-flex>
                </v-layout>

                <template v-if="infrastructureKind === 'openstack'">
                  <v-layout row>
                    <v-flex xs3>
                      <v-select
                      color="cyan"
                      label="Floating Pools"
                      :items="floatingPoolNames"
                      v-model="infrastructureData.floatingPoolName"
                      required
                      ></v-select>
                    </v-flex>
                    <v-flex xs1>
                    </v-flex>
                    <v-flex xs3>
                      <v-select
                      color="cyan"
                      label="Load Balancer Providers"
                      :items="loadBalancerProviderNames"
                      v-model="infrastructureData.loadBalancerProvider"
                      persistent-hint
                      required
                      ></v-select>
                    </v-flex>
                  </v-layout>
                </template>
              </v-card-text>
            </v-container>
          </v-card>
        </v-tab-item>

        <v-tab-item key="worker" id="tab-worker">

          <v-card flat>
            <v-container fluid >
              <transition-group name="list-complete">
                <v-layout row v-for="(worker, index) in workers" :key="worker.id"  class="list-complete-item pt-4 pl-3">
                  <v-flex pa-1 >

                    <worker-input-generic :worker.sync="worker" ref="workerInput"
                      :workers.sync="workers"
                      :cloudProfileName="cloudProfileName"
                      v-if="infrastructureKind === 'aws'">
                      <v-btn v-show="index>0 || workers.length>1"
                        small
                        slot="action"
                        outline
                        icon
                        class="grey--text lighten-2"
                        @click.native.stop="workers.splice(index, 1)">
                        <v-icon>mdi-close</v-icon>
                      </v-btn>
                    </worker-input-generic>

                    <worker-input-generic :worker.sync="worker" ref="workerInput"
                      :workers.sync="workers"
                      :cloudProfileName="cloudProfileName"
                      v-if="infrastructureKind === 'azure'">
                      <v-btn v-show="index>0 || workers.length>1"
                        small
                        slot="action"
                        outline
                        icon
                        class="grey--text lighten-2"
                        @click.native.stop="workers.splice(index, 1)">
                        <v-icon>mdi-close</v-icon>
                      </v-btn>
                    </worker-input-generic>

                    <worker-input-generic :worker.sync="worker" ref="workerInput"
                      :workers.sync="workers"
                      :cloudProfileName="cloudProfileName"
                      v-if="infrastructureKind === 'gcp'">
                      <v-btn v-show="index>0 || workers.length>1"
                        small
                        slot="action"
                        outline
                        icon
                        class="grey--text lighten-2"
                        @click.native.stop="workers.splice(index, 1)">
                        <v-icon>mdi-close</v-icon>
                      </v-btn>
                    </worker-input-generic>

                    <worker-input-openstack :worker.sync="worker" ref="workerInput"
                      :workers.sync="workers"
                      :cloudProfileName="cloudProfileName"
                      v-if="infrastructureKind === 'openstack'">
                      <v-btn v-show="index>0 || workers.length>1"
                        small
                        slot="action"
                        outline
                        icon
                        class="grey--text lighten-2"
                        @click.native.stop="workers.splice(index, 1)">
                        <v-icon>mdi-close</v-icon>
                      </v-btn>
                    </worker-input-openstack>

                  </v-flex>
                </v-layout>
                <v-layout row key="1234" class="list-complete-item pt-4 pl-3 ">

                  <v-flex xs1>
                    <v-btn
                      small
                      @click="addWorker"
                      outline
                      fab
                      icon
                      class="cyan">
                      <v-icon class="cyan--text">add</v-icon>
                    </v-btn>
                  </v-flex>

                  <v-flex xs1 class="mt-2">
                    <v-btn
                      @click="addWorker"
                      flat
                      class="cyan--text">
                      Add Worker Group
                    </v-btn>
                  </v-flex>

                </v-layout>
              </transition-group>
            </v-container>
          </v-card>

        </v-tab-item>

        <v-tab-item key="addons" id="tab-addons">

          <v-card flat>
            <v-container>
              <v-list three-line class="mr-extra">

                <v-list-tile avatar class="list-complete-item">
                  <v-list-tile-action>
                    <v-checkbox color="cyan" v-model="addons['kubernetes-dashboard'].enabled"></v-checkbox>
                  </v-list-tile-action>
                  <v-list-tile-content>
                    <v-list-tile-title >Dashboard</v-list-tile-title>
                    <v-list-tile-sub-title>
                      General-purpose web UI for Kubernetes clusters.
                    </v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>

                <v-list-tile  class="list-complete-item">
                  <v-list-tile-action>
                    <v-checkbox v-model="addons.monocular.enabled" class="cyan--text"></v-checkbox>
                  </v-list-tile-action>
                  <v-list-tile-content>
                    <v-list-tile-title >Monocular</v-list-tile-title>
                    <v-list-tile-sub-title>
                      Monocular is a web-based UI for managing Kubernetes applications and services
                      packaged as Helm Charts. It allows you to search and discover available charts from
                      multiple repositories, and install them in your cluster with one click.
                    </v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>

              </v-list>
            </v-container>
          </v-card>

        </v-tab-item>

        <v-tab-item key="maintenance" id="tab-maintenance">

          <v-card flat>
            <v-container>
              <v-layout row>
                <v-flex xs2>
                  <v-text-field
                   color="cyan"
                   label="Maintenance Start Time"
                   v-model="maintenanceBegin"
                   :error-messages="getErrorMessages('shootDefinition.spec.maintenance.timeWindow.begin')"
                   @input="$v.shootDefinition.spec.maintenance.timeWindow.begin.$touch()"
                   @blur="$v.shootDefinition.spec.maintenance.timeWindow.begin.$touch()"
                   type="time"
                   suffix="UTC"
                   required
                 ></v-text-field>
                </v-flex>
              </v-layout>
              <v-layout row>
                <v-card-title>
                  <span class="subheading">Auto Update</span>
                </v-card-title>
              </v-layout>
              <v-list class="mr-extra">
                <v-list-tile avatar class="list-complete-item">
                  <v-list-tile-action>
                    <v-checkbox color="cyan" v-model="osUpdates" disabled></v-checkbox>
                  </v-list-tile-action>
                  <v-list-tile-content>
                    <v-list-tile-title>Operating System</v-list-tile-title>
                    <v-list-tile-sub-title>
                      Schedule operating system updates of the workers during maintenance.
                    </v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>
                <v-list-tile avatar class="list-complete-item">
                  <v-list-tile-action>
                    <v-checkbox color="cyan" v-model="shootDefinition.spec.maintenance.autoUpdate.kubernetesVersion"></v-checkbox>
                  </v-list-tile-action>
                  <v-list-tile-content>
                    <v-list-tile-title >Kubernetes Patch Version</v-list-tile-title>
                    <v-list-tile-sub-title>
                      Automatically update Kubernetes to latest patch version during maintenance.
                    </v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>
              </v-list>
            </v-container>
          </v-card>

        </v-tab-item>
      </v-tabs-items>
      <alert color="error" :message.sync="errorMessage" :detailedMessage.sync="detailedErrorMessage"></alert>

      <v-card-actions>
        <v-spacer></v-spacer>
        <v-btn flat @click.native.stop="cancelClicked()">Cancel</v-btn>
        <v-btn flat @click.native.stop="createClicked()" :disabled="!valid" class="cyan--text">Create</v-btn>
      </v-card-actions>

    </v-card>
  </v-dialog>
</template>

<script>
  import { mapGetters, mapActions, mapState } from 'vuex'
  import WorkerInputGeneric from '@/components/WorkerInputGeneric'
  import WorkerInputOpenstack from '@/components/WorkerInputOpenstack'
  import CloudProfile from '@/components/CloudProfile'
  import Alert from '@/components/Alert'
  import find from 'lodash/find'
  import random from 'lodash/random'
  import get from 'lodash/get'
  import head from 'lodash/head'
  import sortBy from 'lodash/sortBy'
  import map from 'lodash/map'
  import cloneDeep from 'lodash/cloneDeep'
  import every from 'lodash/every'
  import noop from 'lodash/noop'
  import isEmpty from 'lodash/isEmpty'
  import { required, maxLength } from 'vuelidate/lib/validators'
  import { resourceName, noStartEndHyphen, noConsecutiveHyphen } from '@/utils/validators'
  import CodeBlock from '@/components/CodeBlock'
  import InfraIcon from '@/components/InfrastructureIcon'
  import { setDelayedInputFocus, isOwnSecretBinding, getValidationErrors } from '@/utils'
  import moment from 'moment'

  var semSort = require('semver-sort')

  function shortRandomString (length) {
    const start = 'abcdefghijklmnopqrstuvwxyz'
    const possible = start + '0123456789'
    var text = start.charAt(Math.floor(Math.random() * start.length))
    for (var i = 0; i < (length - 1); i++) {
      text += possible.charAt(Math.floor(Math.random() * possible.length))
    }
    return text
  }

  const validationErrors = {
    shootDefinition: {
      metadata: {
        name: {
          required: 'Name is required',
          maxLength: 'Name ist too long',
          resourceName: 'Name must only be lowercase letters, numbers and hyphens',
          unique: 'Cluster name must be unique',
          noConsecutiveHyphen: 'Cluster name must not contain consecutive hyphens',
          noStartEndHyphen: 'Cluster name must not start or end with a hyphen'
        }
      },
      spec: {
        cloud: {
          secretBindingRef: {
            name: {
              required: 'Secret is required'
            }
          },
          region: {
            required: 'Region is required'
          }
        },
        maintenance: {
          timeWindow: {
            begin: {
              required: 'Maintenance start time is required'
            }
          }
        }
      }
    },
    infrastructureData: {
      zones: {
        zonesNotEmpty: 'Zone is required'
      }
    }
  }

  const defaultShootDefinition = {
    apiVersion: 'garden.sapcloud.io/v1beta1',
    kind: 'Shoot',
    metadata: {
      name: null,
      namespace: null,
      annotations: {
        'garden.sapcloud.io/purpose': 'evaluation'
      }
    },
    spec: {
      cloud: {
        profile: null,
        region: null,
        secretBindingRef: {
          name: null
        }
      },
      kubernetes: {
        version: null
      },
      dns: {
        provider: null,
        domain: null
      },
      maintenance: {
        timeWindow: {
          begin: null,
          end: null
        },
        autoUpdate: {
          kubernetesVersion: true
        }
      },
      addons: {
        'cluster-autoscaler': {
          enabled: true
        },
        heapster: {
          enabled: true
        },
        'kubernetes-dashboard': {
          enabled: true
        },
        monocular: {
          enabled: false
        },
        'nginx-ingress': {
          enabled: true
        }
      }
    }
  }

  export default {
    name: 'create-cluster',
    components: {
      WorkerInputGeneric,
      WorkerInputOpenstack,
      CodeBlock,
      InfraIcon,
      Alert,
      CloudProfile
    },
    props: {
      value: {
        type: Boolean,
        required: false
      }
    },
    data () {
      return {
        shootDefinition: undefined,
        infrastructureData: undefined,
        selectedSecret: undefined,
        selectedInfrastructureKind: undefined,
        activeTab: 'tab-infra',
        purpose: ['evaluation', 'development', 'production'],
        refs_: {},
        validationErrors,
        errorMessage: undefined,
        detailedErrorMessage: undefined,
        osUpdates: true
      }
    },
    validations: {
      shootDefinition: {
        metadata: {
          name: {
            required,
            maxLength: maxLength(10),
            noConsecutiveHyphen,
            noStartEndHyphen,
            resourceName,
            unique (value) {
              return this.shootByNamespaceAndName({namespace: this.namespace, name: value}) === undefined
            }
          }
        },
        spec: {
          cloud: {
            secretBindingRef: {
              name: {
                required
              }
            },
            region: {
              required
            }
          },
          maintenance: {
            timeWindow: {
              begin: {
                required
              }
            }
          }
        }
      },
      infrastructureData: {
        zones: {
          zonesNotEmpty () {
            if (this.infrastructureKind === 'azure') {
              return true
            }
            return !isEmpty(this.infrastructureData.zones)
          }
        }
      }
    },
    computed: {
      ...mapState([
        'user',
        'namespace'
      ]),
      ...mapGetters([
        'cloudProfileByName',
        'machineTypesByCloudProfileName',
        'volumeTypesByCloudProfileName',
        'cloudProfilesByCloudProviderKind',
        'regionsByCloudProfileName',
        'loadBalancerProviderNamesByCloudProfileName',
        'floatingPoolNamesByCloudProfileName',
        'cloudProviderKindList',
        'kubernetesVersions',
        'infrastructureSecretsByInfrastructureKind',
        'infrastructureSecretsByCloudProfileName',
        'projectList',
        'domainList',
        'shootByNamespaceAndName'
      ]),
      visible: {
        get () {
          return this.value
        },
        set (value) {
          this.$emit('input', value)
        }
      },
      clusterName: {
        get () {
          return this.shootDefinition.metadata.name
        },
        set (name) {
          this.shootDefinition.metadata.name = name

          this.setDefaultDomain()
        }
      },
      infrastructureKind: {
        get () {
          return this.selectedInfrastructureKind
        },
        set (infrastructureKind) {
          this.selectedInfrastructureKind = infrastructureKind
          this.infraHandler.setDefaults()

          this.setDefaultCloudProfileName()
        }
      },
      cloudProfileName: {
        get () {
          return this.shootDefinition.spec.cloud.profile
        },
        set (cloudProfileName) {
          this.shootDefinition.spec.cloud.profile = cloudProfileName

          this.setDefaultSecret()
        }
      },
      secret: {
        get () {
          return this.selectedSecret
        },
        set (metadata) {
          const secretBindingRef = {
            name: get(metadata, 'bindingName')
          }
          this.shootDefinition.spec.cloud.secretBindingRef = secretBindingRef

          this.selectedSecret = metadata

          this.setCloudProfileDefaults()
        }
      },
      region: {
        get () {
          return get(this.shootDefinition, 'spec.cloud.region')
        },
        set (region) {
          this.shootDefinition.spec.cloud.region = region

          this.infraHandler.setDefaultZone()
        }
      },
      zone: {
        get () {
          return head(this.infrastructureData.zones)
        },
        set (zone) {
          this.infrastructureData.zones = [zone]
        }
      },
      maintenanceBegin: {
        get () {
          const momentObj = moment.utc(this.shootDefinition.spec.maintenance.timeWindow.begin, 'HHmmZ')
          if (momentObj.isValid()) {
            return momentObj.format('HH:mm:00')
          }
          this.shootDefinition.spec.maintenance.timeWindow.begin = null
          this.shootDefinition.spec.maintenance.timeWindow.end = null
          return null
        },
        set (time) {
          const newMoment = moment.utc(time, 'HHmmZ')
          this.shootDefinition.spec.maintenance.timeWindow.begin = newMoment.format('HHmm00+0000')
          newMoment.add(1, 'h')
          this.shootDefinition.spec.maintenance.timeWindow.end = newMoment.format('HHmm00+0000')
        }
      },
      infrastructure () {
        return this.infrastructureData
      },
      cloudProfiles () {
        return sortBy(this.cloudProfilesByCloudProviderKind(this.infrastructureKind), [(item) => item.metadata.name])
      },
      cloudProfileNames () {
        return map(this.cloudProfiles, 'metadata.name')
      },
      workers () {
        return get(this.infrastructureData, 'workers', [])
      },
      addons () {
        return get(this.shootDefinition, 'spec.addons', {})
      },
      machineTypes () {
        return this.machineTypesByCloudProfileName(this.cloudProfileName)
      },
      volumeTypes () {
        return this.volumeTypesByCloudProfileName(this.cloudProfileName)
      },
      regions () {
        return this.regionsByCloudProfileName(this.cloudProfileName)
      },
      loadBalancerProviderNames () {
        return this.loadBalancerProviderNamesByCloudProfileName(this.cloudProfileName)
      },
      floatingPoolNames () {
        return this.floatingPoolNamesByCloudProfileName(this.cloudProfileName)
      },
      zones () {
        const cloudProfile = this.cloudProfileByName(this.cloudProfileName)
        const predicate = item => item.region === this.region
        return get(find(get(cloudProfile, 'data.zones'), predicate), 'names')
      },
      infrastructureSecretsByKind () {
        return this.infrastructureSecretsByInfrastructureKind(this.infrastructureKind)
      },
      infrastructureSecretsByProfileName () {
        return this.infrastructureSecretsByCloudProfileName(this.cloudProfileName)
      },
      valid () {
        const workerInput = this.refs_.workerInput

        var workersValid = true
        if (workerInput) {
          const isValid = (element, index, array) => {
            return !element.$v.$invalid
          }
          workersValid = every([].concat(workerInput), isValid)
        }
        return workersValid && !this.$v.$invalid
      },
      sortedKubernetesVersions () {
        return semSort.desc(cloneDeep(this.kubernetesVersions(this.cloudProfileName)))
      },
      sortedCloudProviderKindList () {
        return sortBy(this.cloudProviderKindList)
      },
      infraHandler () {
        switch (this.infrastructureKind) {
          case 'aws':
            return {
              setDefaultZone: this.setDefaultZone,
              setDefaults: () => {
                this.infrastructureData = {
                  networks: {
                    vpc: {
                      cidr: '10.250.0.0/16'
                    },
                    internal: [
                      '10.250.112.0/22'
                    ],
                    nodes: '10.250.0.0/16',
                    pods: '100.96.0.0/11',
                    public: [
                      '10.250.96.0/22'
                    ],
                    services: '100.64.0.0/13',
                    workers: [
                      '10.250.0.0/19'
                    ]
                  },
                  workers: null,
                  zones: null
                }
              }
            }
          case 'azure':
            return {
              setDefaultZone: noop,
              setDefaults: () => {
                this.infrastructureData = {
                  networks: {
                    vnet: {
                      cidr: '10.250.0.0/16'
                    },
                    nodes: '10.250.0.0/19',
                    pods: '100.96.0.0/11',
                    services: '100.64.0.0/13',
                    public: '10.250.96.0/22',
                    workers: '10.250.0.0/19'
                  },
                  workers: null
                }
              }
            }
          case 'gcp':
            return {
              setDefaultZone: this.setDefaultZone,
              setDefaults: () => {
                this.infrastructureData = {
                  networks: {
                    nodes: '10.250.0.0/19',
                    pods: '100.96.0.0/11',
                    services: '100.64.0.0/13',
                    workers: [
                      '10.250.0.0/19'
                    ]
                  },
                  workers: null,
                  zones: null
                }
              }
            }
          case 'openstack':
            return {
              setDefaultZone: this.setDefaultZone,
              setDefaults: () => {
                this.infrastructureData = {
                  networks: {
                    nodes: '10.250.0.0/19',
                    pods: '100.96.0.0/11',
                    services: '100.64.0.0/13',
                    workers: [
                      '10.250.0.0/19'
                    ]
                  },
                  workers: null,
                  zones: null
                }
              }
            }
        }
      },
      projectName () {
        const predicate = item => item.metadata.namespace === this.namespace
        const project = find(this.projectList, predicate)
        return get(project, 'metadata.name')
      },
      isOwnSecretBinding () {
        return (secret) => {
          return isOwnSecretBinding(secret)
        }
      }
    },
    methods: {
      ...mapActions([
        'createShoot'
      ]),
      get (object, path, defaultValue) {
        return get(object, path, defaultValue)
      },
      createShootResource () {
        const data = cloneDeep(this.shootDefinition)
        const infrastructureData = cloneDeep(this.infrastructureData)
        infrastructureData.workers.forEach(worker => {
          delete worker.id
        })
        data.spec.cloud[this.infrastructureKind] = infrastructureData
        return this.createShoot(data)
      },
      addWorker () {
        const id = shortRandomString(5)
        this.infrastructureData.workers.push({
          id,
          name: `worker-${id}`,
          machineType: get(head(this.machineTypes), 'name'),
          volumeType: get(head(this.volumeTypes), 'name'),
          volumeSize: '50Gi',
          autoScalerMin: 2,
          autoScalerMax: 2
        })
      },
      createClicked () {
        this.createShootResource()
          .then(() => {
            this.$emit('created')
            this.$emit('close', false)
          })
          .catch(err => {
            const errorCode = get(err, 'response.data.error.code')
            const detailedMessage = get(err, 'response.data.message')
            console.error('Failed to create shoot cluster.', errorCode, detailedMessage, err)
            this.errorMessage = `Failed to create cluster.`
            this.detailedErrorMessage = detailedMessage
          })
      },
      cancelClicked () {
        this.$emit('close', true)
      },
      reset () {
        this.$v.$touch()

        this.activeTab = 'tab-infra'

        this.selectedSecret = undefined
        this.shootDefinition = cloneDeep(defaultShootDefinition)
        this.setDefaultInfrastructureKind()

        this.clusterName = shortRandomString(10)
        this.shootDefinition.metadata.namespace = this.namespace

        const hours = [22, 23, 0, 1, 2, 3, 4, 5]
        const randomHour = get(hours, random(0, hours.length - 1))
        const randomMoment = moment.utc(randomHour, 'HH')
        this.shootDefinition.spec.maintenance.timeWindow.begin = randomMoment.format('HH0000+0000')
        randomMoment.add(1, 'h')
        this.shootDefinition.spec.maintenance.timeWindow.end = randomMoment.format('HH0000+0000')

        this.errorMessage = undefined
        this.detailedMessage = undefined

        setDelayedInputFocus(this, 'name')
      },
      setDefaultDomain () {
        const domain = head(this.domainList)
        this.shootDefinition.spec.dns.domain = `${this.clusterName}.${this.projectName}.${domain.data.domain}`
        this.shootDefinition.spec.dns.provider = domain.data.provider
      },
      setDefaultInfrastructureKind () {
        this.infrastructureKind = head(this.sortedCloudProviderKindList)
      },
      setDefaultCloudProfileName () {
        let cloudProfileName = get(head(this.infrastructureSecretsByKind), 'metadata.cloudProfileName')
        if (!cloudProfileName) {
          cloudProfileName = head(this.cloudProfileNames)
        }
        this.cloudProfileName = cloudProfileName
      },
      setDefaultSecret () {
        this.secret = get(head(this.infrastructureSecretsByProfileName), 'metadata')
      },
      setCloudProfileDefaults () {
        this.setDefaultRegion()
        this.setDefaultKubernetesVersion()
        this.setDefaultWorker()

        if (this.infrastructureKind === 'openstack') {
          this.setDefaultFloatingPoolName()
          this.setDefaultLoadBalancerProvider()
        }
      },
      setDefaultFloatingPoolName () {
        this.infrastructureData.floatingPoolName = head(this.floatingPoolNames)
      },
      setDefaultLoadBalancerProvider () {
        this.infrastructureData.loadBalancerProvider = head(this.loadBalancerProviderNames)
      },
      setDefaultWorker () {
        this.infrastructureData.workers = []
        this.addWorker()
      },
      setDefaultRegion () {
        this.region = head(this.regions)
      },
      setDefaultKubernetesVersion () {
        this.shootDefinition.spec.kubernetes.version = head(this.sortedKubernetesVersions)
      },
      setDefaultZone () {
        const zoneName = head(this.zones)
        if (zoneName) {
          this.infrastructureData.zones = [zoneName]
        } else {
          this.infrastructureData.zones = []
        }
      },
      getErrorMessages (field) {
        return getValidationErrors(this, field)
      }
    },
    watch: {
      value (newValue) {
        if (newValue === true) {
          this.reset()
        }
      }
    },
    created () {
      this.reset()
    },
    mounted () {
      this.refs_ = this.$refs
    }
  }
</script>

<style lang="styl" >

  .dialogContainer{

    .header {
      background-image: url(../assets/cluster_background.svg);
      background-size: cover;
      color:white;
      padding-top:30px;
      .tabs__bar {
        background-color:rgba(255,255,255,0.1)
      }
    }

    .toolbar__title {
      margin-bottom: 30px
      span {
        padding-left: 5px;
      }
    }

    .card__text{
      padding:0;
    }

    .tabs__item {
      font-weight: 400;
    }

    .tabs__items {
      margin: 10px;
      height:410px !important;
      .tabs__content {
        height:100%;
        overflow:auto;
      }
    }

    .input-group{
      label{
        overflow:visible;
      }
    }

    .list-complete-item {
      transition: all 0.5s;
    }

    .add_worker{
      margin-left:30px;
      border:0;
    }

    .list-complete-enter, .list-complete-leave-to {
      opacity: 0;
      transform: translateY(20px);
    }

    .list-complete-leave-active {
      position: absolute;
    }

  }

</style>
