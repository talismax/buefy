<template>
    <b-notification
        v-if="$slots.default != null"
        ref="notification"
        :position="position"
        :model-value="isActive"
        v-bind="$attrs"
        @close="close"
    >
        <slot />
    </b-notification>
    <b-notification
        v-else
        ref="notification"
        :position="position"
        :model-value="isActive"
        v-bind="$attrs"
        @close="close"
    />
</template>

<script>
import config from '../../utils/config'
import NoticeMixinSubset from './NoticeMixinSubset'
import Notification from './Notification'

export default {
    name: 'BNotificationNotice',
    components: {
        [Notification.name]: Notification
    },
    mixins: [NoticeMixinSubset],
    data() {
        return {
            newDuration: this.duration || config.defaultNotificationDuration
        }
    }
}
</script>
