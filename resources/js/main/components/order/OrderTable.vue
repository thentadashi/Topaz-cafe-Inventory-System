<template>
	<a-row>
		<a-col :span="24">
			<div class="table-responsive">
				<a-table
					:columns="columns"
					:row-key="(record) => record.xid"
					:data-source="table.data"
					:pagination="table.pagination"
					:loading="table.loading"
					@change="handleTableChange"
				>
					<template #bodyCell="{ column, record }">
						<template v-if="column.dataIndex === 'order_date'">
							{{ formatDate(record.order_date) }}
						</template>
						<template v-if="column.dataIndex === 'user_id'">
							<user-info :user="record.user" />
						</template>
						<template v-if="column.dataIndex === 'paid_amount'">
							{{ formatAmountCurrency(record.paid_amount) }}
						</template>
						<template v-if="column.dataIndex === 'total_amount'">
							{{ formatAmountCurrency(record.total) }}
						</template>
						<template v-if="column.dataIndex === 'payment_status'">
							<PaymentStatus :paymentStatus="record.payment_status" />
						</template>
						<template v-if="column.dataIndex === 'order_status'">
							<OrderStatus :data="record" />
						</template>
						<template v-if="column.dataIndex === 'action'">
							<a-space
								v-if="
									record.order_type == 'online-orders' &&
									record.order_status != 'delivered' &&
									!record.cancelled
								"
							>
								<a-tooltip
									placement="topLeft"
									:title="$t('stock.view_order')"
								>
									<a-button type="primary" @click="viewOrder(record)">
										<template #icon>
											<EyeOutlined />
										</template>
									</a-button>
								</a-tooltip>
								<a-tooltip
									v-if="
										record.order_status == 'ordered' &&
										!record.cancelled
									"
									placement="topLeft"
									:title="$t('common.confirm')"
								>
									<a-button
										type="primary"
										@click="confirmOrder(record)"
									>
										<template #icon>
											<CheckOutlined />
										</template>
									</a-button>
								</a-tooltip>
								<a-tooltip
									v-if="
										!record.cancelled &&
										record.order_status != 'ordered' &&
										record.order_status != 'delivered'
									"
									placement="topLeft"
									:title="$t('online_orders.confirm_delivery')"
								>
									<a-button
										type="primary"
										@click="confirmDelivery(record)"
									>
										<template #icon>
											<SendOutlined />
										</template>
									</a-button>
								</a-tooltip>
								<a-tooltip
									v-if="
										!record.cancelled &&
										record.order_status != 'delivered'
									"
									placement="topLeft"
									:title="$t('common.cancel')"
								>
									<a-button
										type="primary"
										@click="cancelOrder(record)"
										danger
									>
										<template #icon>
											<StopOutlined />
										</template>
									</a-button>
								</a-tooltip>
							</a-space>
							<a-dropdown v-else placement="bottomRight">
								<MoreOutlined />
								<template #overlay>
									<a-menu>
										<a-menu-item
											key="view"
											v-if="
												permsArray.includes(
													`${pageObject.permission}_view`
												) || permsArray.includes('admin')
											"
											@click="viewItem(record)"
										>
											<EyeOutlined />
											{{ $t("common.view") }}
										</a-menu-item>
										<a-menu-item
											key="edit"
											v-if="
												record.order_type != 'online-orders' &&
												(permsArray.includes(
													`${pageObject.permission}_edit`
												) ||
													permsArray.includes('admin'))
											"
											@click="
												() =>
													$router.push({
														name: `admin.stock.${pageObject.type}.edit`,
														params: {
															id: record.xid,
														},
													})
											"
										>
											<EditOutlined />
											{{ $t("common.edit") }}
										</a-menu-item>
										<a-menu-item
											key="delete"
											v-if="
												record.order_type != 'online-orders' &&
												(permsArray.includes(
													`${pageObject.permission}_delete`
												) ||
													permsArray.includes('admin')) &&
												record.payment_status == 'unpaid'
											"
											@click="showDeleteConfirm(record.xid)"
										>
											<DeleteOutlined />
											{{ $t("common.delete") }}
										</a-menu-item>
										<a-menu-item key="download">
											<!-- <a-typography-link
												:href="`${invoiceBaseUrl}/${record.unique_id}/${selectedLang}`"
												target="_blank"
											> -->
												<DownloadOutlined />
												{{ $t("common.download") }}
											<!-- </a-typography-link> -->
										</a-menu-item>
									</a-menu>
								</template>
							</a-dropdown>
						</template>
					</template>
					<template #expandedRowRender="orderItemData">
						<a-table
							v-if="
								orderItemData &&
								orderItemData.record &&
								orderItemData.record.items
							"
							:row-key="(record) => record.xid"
							:columns="orderItemDetailsColumns"
							:data-source="orderItemData.record.items"
							:pagination="false"
						>
							<template #bodyCell="{ column, record }">
								<template v-if="column.dataIndex === 'product_id'">
									<a-badge>
										<a-avatar
											shape="square"
											:src="record.product.image_url"
										/>
										{{ record.product.name }}
									</a-badge>
								</template>
								<template v-if="column.dataIndex === 'quantity'">
									{{
										`${record.quantity} ${record.product.unit.short_name}`
									}}
								</template>
								<template v-if="column.dataIndex === 'single_unit_price'">
									{{ formatAmountCurrency(record.single_unit_price) }}
								</template>
								<template v-if="column.dataIndex === 'total_discount'">
									{{ formatAmountCurrency(record.total_discount) }}
								</template>
								<template v-if="column.dataIndex === 'total_tax'">
									{{ formatAmountCurrency(record.total_tax) }}
								</template>
								<template v-if="column.dataIndex === 'subtotal'">
									{{ formatAmountCurrency(record.subtotal) }}
								</template>
							</template>
						</a-table>
					</template>
				</a-table>
			</div>
		</a-col>
	</a-row>

	<OrderDetails
		:visible="detailsDrawerVisible"
		:order="selectedItem"
		@close="onDetailDrawerClose"
		@goBack="restSelectedItem"
		@reloadOrder="paymentSuccess"
	/>

	<ConfirmOrder
		:visible="confirmModalVisible"
		:data="modalData"
		@closed="confirmModalVisible = false"
		@confirmSuccess="initialSetup"
	/>

	<ViewOrder
		:visible="viewModalVisible"
		:order="modalData"
		@closed="viewModalVisible = false"
	/>
</template>

<script>
import { onMounted, watch, ref, createVNode } from "vue";
import {
	EyeOutlined,
	PlusOutlined,
	EditOutlined,
	DeleteOutlined,
	ExclamationCircleOutlined,
	MoreOutlined,
	DownloadOutlined,
	CheckOutlined,
	StopOutlined,
	SendOutlined,
} from "@ant-design/icons-vue";
import { Modal, notification } from "ant-design-vue";
import { useRoute } from "vue-router";
import { find } from "lodash-es";
import { useI18n } from "vue-i18n";
import fields from "../../views/stock-management/purchases/fields";
import common from "../../../common/composable/common";
import datatable from "../../../common/composable/datatable";
import PaymentStatus from "../../../common/components/order/PaymentStatus.vue";
import OrderStatus from "../../../common/components/order/OrderStatus.vue";
import Details from "../../views/stock-management/purchases/Details.vue";
import UserInfo from "../../../common/components/user/UserInfo.vue";
import OrderDetails from "./OrderDetails.vue";
import ConfirmOrder from "../../views/stock-management/online-orders/ConfirmOrder.vue";
import ViewOrder from "../../views/stock-management/online-orders/ViewOrder.vue";

export default {
	props: ["orderType", "filters", "extraFilters", "pagination", "perPageItems"],
	components: {
		EyeOutlined,
		PlusOutlined,
		EditOutlined,
		DeleteOutlined,
		MoreOutlined,
		DownloadOutlined,
		ExclamationCircleOutlined,
		CheckOutlined,
		StopOutlined,
		SendOutlined,
		Details,
		UserInfo,
		Details,
		PaymentStatus,
		OrderStatus,
		OrderDetails,

		ConfirmOrder,
		ViewOrder,
	},
	setup(props) {
		const {
			columns,
			hashableColumns,
			setupTableColumns,
			filterableColumns,
			pageObject,
			orderType,
			orderStatus,
			orderItemDetailsColumns,
		} = fields();
		const datatableVariables = datatable();
		const {
			formatAmountCurrency,
			invoiceBaseUrl,
			permsArray,
			calculateOrderFilterString,
			formatDate,
			selectedWarehouse,
			selectedLang,
			orderStatusColors,
		} = common();
		const route = useRoute();
		const { t } = useI18n();
		const detailsDrawerVisible = ref(false);

		const selectedItem = ref({});

		// For Online Orders
		const confirmModalVisible = ref(false);
		const viewModalVisible = ref(false);
		const modalData = ref({});
		// End For Online Orders

		onMounted(() => {
			initialSetup();
		});

		const initialSetup = () => {
			orderType.value = props.orderType;
			if (props.perPageItems) {
				datatableVariables.table.pagination.pageSize = props.perPageItems;
			}
			datatableVariables.table.pagination.current = 1;
			datatableVariables.table.pagination.currentPage = 1;
			datatableVariables.hashable.value = hashableColumns;
			setupTableColumns();
			setUrlData();
		};

		const setUrlData = () => {
			const tableFilter = props.filters;

			const filterString = calculateOrderFilterString(tableFilter);

			datatableVariables.tableUrl.value = {
				url: `${props.orderType}?fields=id,xid,unique_id,invoice_number,order_type,order_date,tax_amount,discount,shipping,subtotal,paid_amount,due_amount,order_status,payment_status,total,tax_rate,staff_user_id,x_staff_user_id,staffMember{id,xid,name,profile_image,profile_image_url,user_type},user_id,x_user_id,user{id,xid,user_type,name,profile_image,profile_image_url,phone},orderPayments{id,xid,amount,payment_id,x_payment_id},orderPayments:payment{id,xid,amount,payment_mode_id,x_payment_mode_id,date,notes},orderPayments:payment:paymentMode{id,xid,name},items{id,xid,product_id,x_product_id,single_unit_price,unit_price,quantity,tax_rate,total_tax,tax_type,total_discount,subtotal},items:product{id,xid,name,image,image_url,unit_id,x_unit_id},items:product:unit{id,xid,name,short_name},items:product:details{id,xid,warehouse_id,x_warehouse_id,product_id,x_product_id,current_stock},cancelled,terms_condition,shippingAddress{id,xid,order_id,name,email,phone,address,shipping_address,city,state,country,zipcode}`,
				filterString,
				filters: {
					user_id: tableFilter.user_id ? tableFilter.user_id : undefined,
				},
				extraFilters: tableFilter.dates ? { dates: tableFilter.dates } : {},
			};
			datatableVariables.table.filterableColumns = filterableColumns;

			if (
				tableFilter.searchColumn &&
				tableFilter.searchString &&
				tableFilter.searchString != ""
			) {
				datatableVariables.table.searchColumn = tableFilter.searchColumn;
				datatableVariables.table.searchString = tableFilter.searchString;
			} else {
				datatableVariables.table.searchColumn = undefined;
				datatableVariables.table.searchString = "";
			}

			datatableVariables.fetch({
				page: 1,
			});
		};

		const showDeleteConfirm = (id) => {
			Modal.confirm({
				title: t("common.delete") + "?",
				icon: createVNode(ExclamationCircleOutlined),
				content: t(`${pageObject.value.langKey}.delete_message`),
				centered: true,
				okText: t("common.yes"),
				okType: "danger",
				cancelText: t("common.no"),
				onOk() {
					axiosAdmin.delete(`${props.orderType}/${id}`).then(() => {
						setUrlData();
						notification.success({
							message: t("common.success"),
							description: t(`${pageObject.value.langKey}.deleted`),
						});
					});
				},
				onCancel() {},
			});
		};

		const viewItem = (record) => {
			selectedItem.value = record;
			detailsDrawerVisible.value = true;
		};

		const restSelectedItem = () => {
			selectedItem.value = {};
		};

		const paymentSuccess = () => {
			datatableVariables.fetch({
				page: datatableVariables.currentPage.value,
				success: (results) => {
					const searchResult = find(results, (result) => {
						return result.xid == selectedItem.value.xid;
					});

					if (searchResult != undefined) {
						selectedItem.value = searchResult;
					}
				},
			});
		};

		const onDetailDrawerClose = () => {
			detailsDrawerVisible.value = false;
		};

		// For Online Orders
		const confirmOrder = (order) => {
			modalData.value = order;
			confirmModalVisible.value = true;
		};

		const viewOrder = (order) => {
			modalData.value = order;
			viewModalVisible.value = true;

			console.log(order);
		};

		const changeOrderStatus = (order) => {
			processRequest({
				url: `online-orders/change-status/${order.unique_id}`,
				data: { order_status: order.order_status },
				success: (res) => {
					// Toastr Notificaiton
					notification.success({
						placement: "bottomRight",
						message: t("common.success"),
						description: t("online_orders.order_status_changed"),
					});
				},
				error: (errorRules) => {},
			});
		};

		const cancelOrder = (order) => {
			Modal.confirm({
				title: t("online_orders.cancel_order") + "?",
				icon: createVNode(ExclamationCircleOutlined),
				content: t(`online_orders.cancel_message`),
				centered: true,
				okText: t("common.yes"),
				okType: "danger",
				cancelText: t("common.no"),
				onOk() {
					axiosAdmin
						.post(`online-orders/cancel/${order.unique_id}`)
						.then(() => {
							initialSetup();
							notification.success({
								message: t("common.success"),
								description: t(`online_orders.order_cancelled`),
								placement: "bottomRight",
							});
						});
				},
				onCancel() {},
			});
		};

		const confirmDelivery = (order) => {
			Modal.confirm({
				title: t("common.delivered") + "?",
				icon: createVNode(ExclamationCircleOutlined),
				content: t(`online_orders.deliver_message`),
				centered: true,
				okText: t("common.yes"),
				okType: "danger",
				cancelText: t("common.no"),
				onOk() {
					axiosAdmin
						.post(`online-orders/delivered/${order.unique_id}`)
						.then(() => {
							initialSetup();
							notification.success({
								message: t("common.success"),
								description: t(`online_orders.order_delivered`),
								placement: "bottomRight",
							});
						});
				},
				onCancel() {},
			});
		};
		// End For Online Orders

		watch(props, (newVal, oldVal) => {
			initialSetup();
			restSelectedItem();
		});

		watch(selectedWarehouse, (newVal, oldVal) => {
			setUrlData();
		});

		return {
			columns,
			...datatableVariables,
			filterableColumns,
			pageObject,

			formatDate,
			orderStatus,
			orderStatusColors,

			setUrlData,
			formatAmountCurrency,
			invoiceBaseUrl,
			permsArray,

			selectedItem,
			viewItem,
			restSelectedItem,
			paymentSuccess,

			showDeleteConfirm,

			detailsDrawerVisible,
			onDetailDrawerClose,
			orderItemDetailsColumns,
			selectedLang,
			initialSetup,

			// For Online Orders
			confirmOrder,
			cancelOrder,
			viewOrder,
			confirmDelivery,
			changeOrderStatus,
			confirmModalVisible,
			viewModalVisible,
			modalData,
			// End For Online Orders
		};
	},
};
</script>
