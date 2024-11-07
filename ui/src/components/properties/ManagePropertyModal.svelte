<script lang="ts">
	import FormWrapperDropdown from '@components/generic/FormWrapperDropdown.svelte'
	import SetNotSetIndicator from '@components/generic/SetNotSetIndicator.svelte'
	import {
		TEMP_HIDE,
		PROPERTIES,
		SHELLS,
		REALTOR_GRADE,
		CONFIG,
	} from '@store/stores'
	import { ReceiveNUI } from '@utils/ReceiveNUI'
	import type { IProperty } from '@typings/type'
	import { SendNUI } from '@utils/SendNUI'
	import { createEventDispatcher } from 'svelte'
	import { fade } from 'svelte/transition'

	const dispatch = createEventDispatcher()

	export let manageProperty: boolean = false,
		selectedProperty: IProperty | null = null

	const index = $PROPERTIES.findIndex(
		(property) => property.property_id === selectedProperty.property_id
	)

	let forSaleDropDownValues = ['Á venda', 'Indisponível'],
		selectedForSaleDropdownValue = selectedProperty.for_sale
			? forSaleDropDownValues[0]
			: forSaleDropDownValues[1]

	function updateForSaleDropdownValue(value) {
		const isForSale = value === forSaleDropDownValues[0] ? true : false
		SendNUI('updatePropertyData', {
			type: 'UpdateForSale',
			property_id: selectedProperty.property_id,
			data: { forsale: isForSale },
		})
		$PROPERTIES[index].for_sale = isForSale ? 1 : 0
		selectedProperty.for_sale = isForSale ? 1 : 0
		// selectedForSaleDropdownValue = value;
	}

	let finalizedOwner = selectedProperty.owner ? selectedProperty.owner : ''

	let description = selectedProperty.description ?? ''

	let propertyPrice = selectedProperty.price

	let newShell = selectedProperty.shell

	function updatePropertyValues(typeUpdate, dataObject, key, value) {
		SendNUI('updatePropertyData', {
			type: typeUpdate,
			property_id: selectedProperty.property_id,
			data: dataObject,
		})
		$PROPERTIES[index][key] = value
		selectedProperty[key] = value
	}

	let doorValueSet = selectedProperty.door_data.length > 0 ? true : false
	let garageValueSet = selectedProperty.garage_data
		? selectedProperty.garage_data.x
			? true
			: false
		: false

	function handleZonePlacement(type) {
		SendNUI('startZonePlacement', {
			type: type,
			property_id: selectedProperty.property_id,
		}).then(() => {
			$TEMP_HIDE = true
		})
	}

	let propertyImages = selectedProperty.extra_imgs,
		newImageName = '',
		newImageUrl = ''

	function addNewImage() {
		propertyImages = [
			...propertyImages,
			{
				label: newImageName,
				url: newImageUrl,
			},
		]
		newImageName = ''
		newImageUrl = ''

		updatePropertyValues(
			'UpdateImgs',
			{ imgs: propertyImages },
			'extra_imgs',
			propertyImages
		)
	}

	function deleteProperty() {
		dispatch('delete-property', selectedProperty)
	}

	ReceiveNUI('garageMade', () => {
		garageValueSet = true
	})
</script>

<div
	class="modal large-footer-modal"
	tabindex="-1"
	aria-hidden="true"
	transition:fade={{ duration: 100 }}
>
	<div
		class="modal-dialog large-footer-modal-dialog manage-property-modal-dialog"
	>
		<div class="modal-content large-footer-modal-content">
			<div class="modal-body large-footer-modal-body">
				<div class="header">
					<div class="heading-title-wrapper">
						<i class="fas fa-pen info-icon" />
						<p>Gerenciar Propriedade</p>
					</div>
					<div on:click={() => (manageProperty = false)}>
						<i class="fas fa-xmark close-icon" />
					</div>
				</div>

				<div
					class="large-footer-modal-body-data manage-property-large-footer-modal-body-data"
				>
					<div class="data-details-manage-property">
						<div class="left-column">
							<p class="heading">Descrição ao vivo</p>
							<p class="info">
								Altere as configurações após a criação!
							</p>
						</div>

						<div class="right-column">
							{#if $REALTOR_GRADE >= $CONFIG.changePropertyForSale}
								<div
									id="sell-property"
									class="form-row-wrapper"
								>
									<p class="label">Alterar Status da Propriedade</p>

									<div class="action-row">
										<SetNotSetIndicator
											rightValue={selectedProperty.for_sale
												? 'Definido'
												: 'Não definido'}
											leftValue={'Status'}
											good={selectedProperty.for_sale}
										/>

										<div style="margin-left: 0.5vw;">
											<FormWrapperDropdown
												dropdownValues={forSaleDropDownValues}
												label=""
												insideLabel="Alterar: "
												selectedValue={selectedForSaleDropdownValue}
												on:selected-dropdown={(event) =>
													updateForSaleDropdownValue(
														event.detail
													)}
											/>
										</div>
									</div>
								</div>
							{/if}

							{#if $REALTOR_GRADE >= $CONFIG.sellProperty && selectedProperty.for_sale == 1}
								<div
									id="finalize-sell-property"
									class="form-row-wrapper"
								>
									<p class="label">Finalizar Venda da Propriedade</p>

									<div class="action-row">
										<SetNotSetIndicator
											leftValue={finalizedOwner?.trim() !== '' ? 'Definido' : 'Não definido'}
											rightValue=""
											good={finalizedOwner?.trim() !== ''}
										/>
										<input
											type="text"
											placeholder="ID: 34343434343"
											style="width: 10vw;"
											bind:value={finalizedOwner}
										/>
										<button
											class="regular-button"
											on:click={() =>
												updatePropertyValues(
													'UpdateOwner',
													{
														targetSrc:
															finalizedOwner,
													},
													'owner',
													finalizedOwner
												)}>Request</button
										>
									</div>
								</div>
							{/if}

		                    {#if $REALTOR_GRADE >= $CONFIG.manageProperty}
							    <div
							    	id="manage-description"
							    	class="form-row-wrapper"
							    >
							    	<p class="label">Alterar Descrição</p>
    
							    	<div class="action-row">
							    		<textarea
							    			rows="3"
							    			placeholder="Escreva uma breve e encantadora descrição sobre a propriedade..."
							    			style="width: 18vw;"
							    			bind:value={description}
							    			on:keyup={() =>
							    				updatePropertyValues(
							    					'UpdateDescription',
							    					{ description: description },
							    					'description',
							    					description
							    				)}
							    		/>
							    	</div>
							    </div>

							    <div id="manage-price" class="form-row-wrapper">
							    	<p class="label">Alterar Preço (R$)</p>
    
							    	<div class="action-row">
							    		<input
							    			type="number"
							    			placeholder="$1000000000"
							    			style="width: 10vw;"
							    			bind:value={propertyPrice}
							    			on:keyup={() =>
							    				updatePropertyValues(
							    					'UpdatePrice',
							    					{ price: propertyPrice },
							    					'price',
							    					propertyPrice
							    				)}
							    		/>
							    	</div>
							    </div>

		                        {#if selectedProperty.shell !== 'mlo'}
							    <div
							    	id="manage-shell-type"
							    	class="form-row-wrapper"
							    >
							    	<p class="label">Alterar Shell (Interior)</p>
    
							    	<div class="action-row">
							    		<FormWrapperDropdown
							    			dropdownValues={Object.keys($SHELLS)}
							    			label=""
							    			id="manage-dd-shell"
							    			selectedValue={newShell}
							    			insideLabel="Type: "
							    			on:selected-dropdown={(event) => {
							    				newShell = event.detail
							    				updatePropertyValues(
							    					'UpdateShell',
							    					{ shell: newShell },
							    					'shell',
							    					newShell
							    				)
							    			}}
							    		/>
							    	</div>
							    </div>
							    {/if}
    
							    <div
							    	id="add-images"
							    	class="form-row-wrapper"
							    	style="margin-top: 2vw"
							    >
							    	<p class="label">Adicionar Imagens</p>
    
							    	<div class="action-row">
							    		<input
							    			id="img-name"
							    			type="text"
							    			placeholder="Nome"
							    			style="width: 7vw;"
							    			bind:value={newImageName}
							    		/>
							    		<input
							    			id="img-url"
							    			type="text"
							    			placeholder="URL/Link"
							    			style="width: 7vw;"
							    			bind:value={newImageUrl}
							    		/>
							    		<button
							    			class="regular-button"
							    			on:click={addNewImage}>Add</button
							    		>
							    	</div>
    
							    	<div class="image-tiles-wrapper">
							    		{#each propertyImages as image, index}
							    			<div>
							    				<img src={image.url} alt="" />
							    			</div>
							    		{/each}
							    	</div>
							    </div>
    
    
		                        {#if selectedProperty.shell !== 'mlo'}
							        <div id="manage-door" class="form-row-wrapper">
							        	<p class="label">Gerenciar Porta</p>
    
							        	<div class="action-row">
							        		<SetNotSetIndicator
							        			leftValue="Porta"
							        			rightValue={doorValueSet
							        				? 'Definido'
							        				: 'Não definido'}
							        			good={doorValueSet}
							        		/>
							        		<button
							        			class="regular-button"
							        			on:click={() =>
							        				handleZonePlacement('door')}
							        			>Nova localização</button
							        		>
							        		<button class="disable-button"
							        			>Remover</button
							        		>
							        	</div>
							        </div>
							    {/if}
    
							    <div id="manage-garage" class="form-row-wrapper">
							    	<p class="label">Gerenciar Garagem</p>

							    	<div class="action-row">
							    		<SetNotSetIndicator
							    			leftValue="Garagem"
							    			rightValue={garageValueSet
							    				? 'Definido'
							    				: 'Não definido'}
							    			good={garageValueSet}
							    		/>
							    		<button
							    			class="regular-button"
							    			on:click={() =>
							    				handleZonePlacement('garage')}
							    			>Nova Localização</button
							    		>
							    		<button
							    			class="disable-button"
							    			on:click={() =>
							    				updatePropertyValues(
							    					'UpdateGarage',
							    					{},
							    					'garage_data',
							    					null
							    				)}>Remover</button
							    		>
							    	</div>
							    </div>
							{/if}
						</div>
					</div>
				</div>

				<div class="large-footer-modal-footer-manage-property">
					{#if $REALTOR_GRADE >= $CONFIG.deleteProperty}
						<button class="delete-button" on:click={deleteProperty}>
							Excluir Propriedade
						</button>
					{/if}
				</div>
			</div>
		</div>
	</div>
</div>
