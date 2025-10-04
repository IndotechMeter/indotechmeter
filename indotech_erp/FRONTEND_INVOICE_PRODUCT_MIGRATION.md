# Frontend Invoice-Product Integration Migration Guide

## Executive Summary
This guide details how to migrate the existing frontend invoice structure to integrate with the new Products API, focusing on invoice creation and viewing functionality.

## Current vs New Architecture

### Current Structure (Manual Entry)
```typescript
// Current InvoiceItem - Manual entry
interface InvoiceItem {
  id: string;
  description: string;      // Manually typed
  hsnCode?: string;         // Manually entered
  quantity: number;         // Manually entered
  rate: number;             // Manually entered
  total: number;            // Calculated
  unit?: string;            // Manually selected
  // Tax calculations...
}
```

### New Structure (Product-Integrated)
```typescript
// New InvoiceItem - Product-linked
interface InvoiceItem {
  id: string;
  productId: string;        // NEW: Links to product catalog
  productSnapshot: {        // NEW: Preserves product state at invoice time
    sku: string;
    name: string;
    description: string;
    hsnCode: string;
    basePrice: number;
    unit: string;
    stockLevel: number;
    category: string;
    taxRate: number;
  };

  // User-modifiable fields
  description: string;      // Auto-filled from product, can override
  hsnCode: string;          // Auto-filled from product
  quantity: number;         // User input with stock validation
  rate: number;             // Auto-filled, can override with tracking
  priceOverride?: {         // NEW: Track price changes
    originalPrice: number;
    overridePrice: number;
    discountPercentage: number;
    reason?: string;
  };

  // Calculated fields
  total: number;
  stockWarning?: boolean;   // NEW: Low stock indicator
  // Tax calculations remain same...
}
```

## Migration Steps

### Phase 1: Update Type Definitions

**File:** `frontend/src/modules/billing/types/invoice.types.ts`

```typescript
// Add new fields while maintaining backward compatibility
export interface InvoiceItem {
  id: string;

  // Product integration fields (optional for backward compatibility)
  productId?: string;
  productSnapshot?: ProductSnapshot;
  priceOverride?: PriceOverride;
  stockWarning?: boolean;

  // Existing fields remain
  description: string;
  hsnCode?: string;
  quantity: number;
  rate: number;
  total: number;
  unit?: string;
  isService?: boolean;

  // Tax fields remain unchanged
  taxableRate?: number;
  totalTaxable?: number;
  totalTax?: number;
  gstRate?: number;
}

interface ProductSnapshot {
  sku: string;
  name: string;
  description: string;
  hsnCode: string;
  basePrice: number;
  unit: string;
  stockLevel: number;
  category: string;
  taxRate: number;
}

interface PriceOverride {
  originalPrice: number;
  overridePrice: number;
  discountPercentage: number;
  reason?: string;
}
```

### Phase 2: Create Product Search Component

**New File:** `frontend/src/modules/billing/components/invoice-form/ProductSearchField.tsx`

```typescript
import React, { useState, useCallback } from 'react';
import { useQuery } from '@tanstack/react-query';
import { debounce } from 'lodash';
import { Search, Package, AlertCircle } from 'lucide-react';
import { Input } from '@/components/ui/input';
import {
  Command,
  CommandEmpty,
  CommandGroup,
  CommandInput,
  CommandItem,
  CommandList,
} from '@/components/ui/command';
import {
  Popover,
  PopoverContent,
  PopoverTrigger,
} from '@/components/ui/popover';
import { Badge } from '@/components/ui/badge';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { formatCurrency } from '../../utils/billingFormatters';

interface ProductSearchFieldProps {
  onProductSelect: (product: Product) => void;
  disabled?: boolean;
}

interface Product {
  productId: string;
  sku: string;
  name: string;
  description: string;
  hsnCode: string;
  basePrice: number;
  unit: string;
  stockLevel: number;
  category: string;
  taxRate: number;
}

const ProductSearchField: React.FC<ProductSearchFieldProps> = ({
  onProductSelect,
  disabled = false,
}) => {
  const [searchTerm, setSearchTerm] = useState('');
  const [isOpen, setIsOpen] = useState(false);

  // Debounced search query
  const { data: products, isLoading, error } = useQuery({
    queryKey: ['products', 'search', searchTerm],
    queryFn: async () => {
      if (searchTerm.length < 2) return [];

      const response = await fetch(
        `/v1/products/invoice/search?q=${encodeURIComponent(searchTerm)}`
      );

      if (!response.ok) {
        throw new Error('Failed to search products');
      }

      return response.json();
    },
    enabled: searchTerm.length >= 2,
    staleTime: 30000, // Cache for 30 seconds
  });

  // Debounced search handler
  const debouncedSearch = useCallback(
    debounce((value: string) => {
      setSearchTerm(value);
    }, 300),
    []
  );

  const handleSelect = (product: Product) => {
    onProductSelect(product);
    setIsOpen(false);
    setSearchTerm('');
  };

  const getStockBadge = (stockLevel: number) => {
    if (stockLevel <= 0) {
      return <Badge variant="destructive">Out of Stock</Badge>;
    }
    if (stockLevel < 10) {
      return <Badge variant="warning">Low Stock ({stockLevel})</Badge>;
    }
    return <Badge variant="success">In Stock ({stockLevel})</Badge>;
  };

  return (
    <Popover open={isOpen} onOpenChange={setIsOpen}>
      <PopoverTrigger asChild>
        <div className="relative">
          <Search className="absolute left-2 top-2.5 h-4 w-4 text-muted-foreground" />
          <Input
            placeholder="Search products by name, SKU, or category..."
            className="pl-8"
            onChange={(e) => debouncedSearch(e.target.value)}
            onFocus={() => setIsOpen(true)}
            disabled={disabled}
          />
        </div>
      </PopoverTrigger>

      <PopoverContent className="w-[600px] p-0" align="start">
        <Command>
          <CommandList>
            {isLoading && (
              <CommandEmpty>Searching products...</CommandEmpty>
            )}

            {error && (
              <CommandEmpty>
                <Alert variant="destructive">
                  <AlertCircle className="h-4 w-4" />
                  <AlertDescription>
                    Failed to search products. Please try again.
                  </AlertDescription>
                </Alert>
              </CommandEmpty>
            )}

            {!isLoading && !error && products?.length === 0 && searchTerm.length >= 2 && (
              <CommandEmpty>No products found.</CommandEmpty>
            )}

            {!isLoading && !error && searchTerm.length < 2 && (
              <CommandEmpty>Type at least 2 characters to search...</CommandEmpty>
            )}

            {products && products.length > 0 && (
              <CommandGroup heading="Products">
                {products.map((product: Product) => (
                  <CommandItem
                    key={product.productId}
                    onSelect={() => handleSelect(product)}
                    className="cursor-pointer"
                  >
                    <div className="flex items-start justify-between w-full">
                      <div className="flex-1">
                        <div className="flex items-center gap-2">
                          <Package className="h-4 w-4 text-muted-foreground" />
                          <span className="font-medium">{product.name}</span>
                          <span className="text-xs text-muted-foreground">
                            SKU: {product.sku}
                          </span>
                        </div>
                        <div className="text-sm text-muted-foreground mt-1">
                          {product.description}
                        </div>
                        <div className="flex gap-4 mt-1 text-xs">
                          <span>HSN: {product.hsnCode}</span>
                          <span>Unit: {product.unit}</span>
                          <span>Category: {product.category}</span>
                        </div>
                      </div>
                      <div className="text-right ml-4">
                        <div className="font-semibold">
                          {formatCurrency(product.basePrice)}
                        </div>
                        <div className="mt-1">
                          {getStockBadge(product.stockLevel)}
                        </div>
                      </div>
                    </div>
                  </CommandItem>
                ))}
              </CommandGroup>
            )}
          </CommandList>
        </Command>
      </PopoverContent>
    </Popover>
  );
};

export default ProductSearchField;
```

### Phase 3: Update LineItemsSection Component

**File:** `frontend/src/modules/billing/components/invoice-form/LineItemsSection.tsx`

```typescript
import React, { useCallback, useState } from 'react';
import { useFormContext, useFieldArray, Controller } from 'react-hook-form';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Trash2, Plus, AlertTriangle, Package } from 'lucide-react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Alert, AlertDescription, AlertTitle } from '@/components/ui/alert';
import { Badge } from '@/components/ui/badge';
import {
  Tooltip,
  TooltipContent,
  TooltipProvider,
  TooltipTrigger,
} from '@/components/ui/tooltip';
import { formatCurrency } from '../../utils/billingFormatters';
import ProductSearchField from './ProductSearchField';

// Feature flag for product integration
const ENABLE_PRODUCT_INTEGRATION = process.env.REACT_APP_PRODUCT_INTEGRATION !== 'false';
const STRICT_MODE = process.env.REACT_APP_STRICT_MODE === 'true';

const LineItemsSection: React.FC = () => {
  const { control, watch, setValue } = useFormContext();
  const { fields, append, remove, update } = useFieldArray({
    control,
    name: 'items',
  });

  const [stockWarnings, setStockWarnings] = useState<Record<string, boolean>>({});
  const [priceOverrides, setPriceOverrides] = useState<Record<string, boolean>>({});

  const gstRate = watch('gst.rate');
  const numericGstRate = parseFloat(gstRate?.replace('%', '') || '0');

  // Handle product selection from search
  const handleProductSelect = useCallback((product: any, index?: number) => {
    const newItem = {
      id: `item-${Date.now()}`,
      productId: product.productId,
      productSnapshot: {
        sku: product.sku,
        name: product.name,
        description: product.description,
        hsnCode: product.hsnCode,
        basePrice: product.basePrice,
        unit: product.unit,
        stockLevel: product.stockLevel,
        category: product.category,
        taxRate: product.taxRate,
      },
      description: product.description,
      hsnCode: product.hsnCode,
      quantity: 1,
      rate: product.basePrice,
      unit: product.unit,
      total: product.basePrice,
      // Tax calculations
      taxableRate: product.basePrice / (1 + numericGstRate / 100),
      totalTaxable: product.basePrice / (1 + numericGstRate / 100),
      totalTax: (product.basePrice / (1 + numericGstRate / 100)) * (numericGstRate / 100),
      gstRate: numericGstRate,
    };

    if (index !== undefined) {
      // Replace existing item
      update(index, newItem);
    } else {
      // Add new item
      append(newItem);
    }

    // Check stock level
    if (product.stockLevel < 10) {
      setStockWarnings(prev => ({ ...prev, [newItem.id]: true }));
    }
  }, [append, update, numericGstRate]);

  // Handle quantity change with stock validation
  const handleQuantityChange = useCallback((index: number, value: number) => {
    const item = watch(`items.${index}`);

    // Update quantity
    setValue(`items.${index}.quantity`, value);

    // Recalculate totals
    const total = value * (item.rate || 0);
    setValue(`items.${index}.total`, total);

    // Stock validation if product-linked
    if (ENABLE_PRODUCT_INTEGRATION && item.productSnapshot) {
      const stockLevel = item.productSnapshot.stockLevel;

      if (value > stockLevel) {
        setStockWarnings(prev => ({ ...prev, [item.id]: true }));

        if (STRICT_MODE) {
          // In strict mode, prevent over-ordering
          setValue(`items.${index}.quantity`, stockLevel);
          alert(`Cannot order more than available stock (${stockLevel} units)`);
        }
      } else if (stockLevel - value < 10) {
        setStockWarnings(prev => ({ ...prev, [item.id]: true }));
      } else {
        setStockWarnings(prev => ({ ...prev, [item.id]: false }));
      }
    }

    // Recalculate tax
    if (numericGstRate > 0) {
      const taxableRate = item.rate / (1 + numericGstRate / 100);
      const totalTaxable = taxableRate * value;
      const totalTax = totalTaxable * (numericGstRate / 100);

      setValue(`items.${index}.taxableRate`, taxableRate);
      setValue(`items.${index}.totalTaxable`, totalTaxable);
      setValue(`items.${index}.totalTax`, totalTax);
    }
  }, [setValue, watch, numericGstRate]);

  // Handle rate change with price override tracking
  const handleRateChange = useCallback((index: number, value: number) => {
    const item = watch(`items.${index}`);

    setValue(`items.${index}.rate`, value);

    // Track price override if product-linked
    if (ENABLE_PRODUCT_INTEGRATION && item.productSnapshot) {
      const originalPrice = item.productSnapshot.basePrice;
      const priceDifference = Math.abs(value - originalPrice);
      const percentageChange = (priceDifference / originalPrice) * 100;

      if (percentageChange > 10) {
        setPriceOverrides(prev => ({ ...prev, [item.id]: true }));

        // Store price override details
        setValue(`items.${index}.priceOverride`, {
          originalPrice,
          overridePrice: value,
          discountPercentage: ((originalPrice - value) / originalPrice) * 100,
          reason: '', // Can be filled by user
        });
      } else {
        setPriceOverrides(prev => ({ ...prev, [item.id]: false }));
        setValue(`items.${index}.priceOverride`, undefined);
      }
    }

    // Recalculate totals
    const quantity = item.quantity || 1;
    const total = quantity * value;
    setValue(`items.${index}.total`, total);

    // Recalculate tax
    if (numericGstRate > 0) {
      const taxableRate = value / (1 + numericGstRate / 100);
      const totalTaxable = taxableRate * quantity;
      const totalTax = totalTaxable * (numericGstRate / 100);

      setValue(`items.${index}.taxableRate`, taxableRate);
      setValue(`items.${index}.totalTaxable`, totalTaxable);
      setValue(`items.${index}.totalTax`, totalTax);
    }
  }, [setValue, watch, numericGstRate]);

  // Add empty line item (manual or product search)
  const handleAddItem = useCallback(() => {
    if (!ENABLE_PRODUCT_INTEGRATION) {
      // Traditional manual entry
      append({
        id: `item-${Date.now()}`,
        description: '',
        hsnCode: '',
        quantity: 1,
        rate: 0,
        unit: 'NOS',
        total: 0,
      });
    }
    // If product integration is enabled, the ProductSearchField handles adding
  }, [append]);

  return (
    <Card>
      <CardHeader className="pb-3">
        <div className="flex justify-between items-center">
          <CardTitle>Line Items</CardTitle>
          <div className="flex gap-2">
            {ENABLE_PRODUCT_INTEGRATION && (
              <ProductSearchField
                onProductSelect={(product) => handleProductSelect(product)}
                disabled={false}
              />
            )}
            {!ENABLE_PRODUCT_INTEGRATION && (
              <Button
                type="button"
                onClick={handleAddItem}
                size="sm"
                className="flex items-center gap-1"
              >
                <Plus className="h-4 w-4" /> Add Item
              </Button>
            )}
          </div>
        </div>
      </CardHeader>

      <CardContent>
        {fields.length === 0 ? (
          <Alert className="mb-4">
            <AlertTitle>No items added</AlertTitle>
            <AlertDescription>
              {ENABLE_PRODUCT_INTEGRATION
                ? 'Search and select products to add to this invoice.'
                : 'Add at least one item to this invoice.'}
            </AlertDescription>
          </Alert>
        ) : (
          <div className="space-y-6">
            {/* Header */}
            <div className="grid grid-cols-12 gap-3 text-sm font-medium text-muted-foreground mb-1">
              {ENABLE_PRODUCT_INTEGRATION && (
                <div className="col-span-1">Product</div>
              )}
              <div className={ENABLE_PRODUCT_INTEGRATION ? "col-span-3" : "col-span-4"}>
                Description
              </div>
              <div className="col-span-2">HSN Code</div>
              <div className="col-span-2 text-right">Qty</div>
              <div className="col-span-2 text-right">Rate</div>
              <div className="col-span-2 text-right">Total</div>
            </div>

            {/* Line Items */}
            {fields.map((field, index) => {
              const item = watch(`items.${index}`);
              const hasStockWarning = stockWarnings[item.id];
              const hasPriceOverride = priceOverrides[item.id];

              return (
                <div
                  key={field.id}
                  className={`grid grid-cols-12 gap-3 p-3 rounded-lg border ${
                    hasStockWarning ? 'border-yellow-500 bg-yellow-50' : 'border-gray-200'
                  }`}
                >
                  {/* Product Info Badge */}
                  {ENABLE_PRODUCT_INTEGRATION && (
                    <div className="col-span-1 flex items-center">
                      {item.productSnapshot ? (
                        <TooltipProvider>
                          <Tooltip>
                            <TooltipTrigger>
                              <Badge variant="secondary" className="cursor-help">
                                <Package className="h-3 w-3 mr-1" />
                                {item.productSnapshot.sku}
                              </Badge>
                            </TooltipTrigger>
                            <TooltipContent>
                              <div className="text-xs">
                                <p>Product: {item.productSnapshot.name}</p>
                                <p>Category: {item.productSnapshot.category}</p>
                                <p>Stock: {item.productSnapshot.stockLevel} units</p>
                              </div>
                            </TooltipContent>
                          </Tooltip>
                        </TooltipProvider>
                      ) : (
                        <Badge variant="outline">Manual</Badge>
                      )}
                    </div>
                  )}

                  {/* Description */}
                  <div className={ENABLE_PRODUCT_INTEGRATION ? "col-span-3" : "col-span-4"}>
                    <Controller
                      name={`items.${index}.description`}
                      control={control}
                      render={({ field }) => (
                        <Input
                          {...field}
                          placeholder="Item description"
                          className={item.productSnapshot ? 'bg-gray-50' : ''}
                        />
                      )}
                    />
                  </div>

                  {/* HSN Code */}
                  <div className="col-span-2">
                    <Controller
                      name={`items.${index}.hsnCode`}
                      control={control}
                      render={({ field }) => (
                        <Input
                          {...field}
                          placeholder="HSN Code"
                          className={item.productSnapshot ? 'bg-gray-50' : ''}
                        />
                      )}
                    />
                  </div>

                  {/* Quantity with Stock Warning */}
                  <div className="col-span-2">
                    <div className="relative">
                      <Controller
                        name={`items.${index}.quantity`}
                        control={control}
                        render={({ field }) => (
                          <Input
                            {...field}
                            type="number"
                            min="1"
                            className="text-right"
                            onChange={(e) => handleQuantityChange(index, parseFloat(e.target.value) || 0)}
                          />
                        )}
                      />
                      {hasStockWarning && (
                        <TooltipProvider>
                          <Tooltip>
                            <TooltipTrigger className="absolute right-1 top-1">
                              <AlertTriangle className="h-4 w-4 text-yellow-600" />
                            </TooltipTrigger>
                            <TooltipContent>
                              <p>Low stock or exceeds available quantity</p>
                            </TooltipContent>
                          </Tooltip>
                        </TooltipProvider>
                      )}
                    </div>
                  </div>

                  {/* Rate with Price Override Indicator */}
                  <div className="col-span-2">
                    <div className="relative">
                      <Controller
                        name={`items.${index}.rate`}
                        control={control}
                        render={({ field }) => (
                          <Input
                            {...field}
                            type="number"
                            step="0.01"
                            className={`text-right ${hasPriceOverride ? 'border-blue-500' : ''}`}
                            onChange={(e) => handleRateChange(index, parseFloat(e.target.value) || 0)}
                          />
                        )}
                      />
                      {hasPriceOverride && (
                        <Badge
                          variant="secondary"
                          className="absolute -top-2 -right-2 text-xs"
                        >
                          Modified
                        </Badge>
                      )}
                    </div>
                  </div>

                  {/* Total */}
                  <div className="col-span-2 flex items-center justify-between">
                    <span className="text-right font-medium flex-1">
                      {formatCurrency(item.total || 0)}
                    </span>
                    <Button
                      type="button"
                      variant="ghost"
                      size="sm"
                      onClick={() => remove(index)}
                      className="ml-2"
                    >
                      <Trash2 className="h-4 w-4" />
                    </Button>
                  </div>
                </div>
              );
            })}

            {/* Manual Add Button (when product integration is enabled) */}
            {ENABLE_PRODUCT_INTEGRATION && (
              <div className="flex justify-center pt-4 border-t">
                <Button
                  type="button"
                  variant="outline"
                  size="sm"
                  onClick={handleAddItem}
                  className="text-xs"
                >
                  <Plus className="h-3 w-3 mr-1" />
                  Add Manual Entry
                </Button>
              </div>
            )}
          </div>
        )}
      </CardContent>
    </Card>
  );
};

export default LineItemsSection;
```

### Phase 4: Update Invoice View Components

**File:** `frontend/src/modules/billing/pages/invoice-detail/InvoiceItemsTable.tsx`

```typescript
import React from 'react';
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table';
import { Badge } from '@/components/ui/badge';
import { Package, AlertTriangle } from 'lucide-react';
import {
  Tooltip,
  TooltipContent,
  TooltipProvider,
  TooltipTrigger,
} from '@/components/ui/tooltip';
import { formatCurrency } from '../../utils/billingFormatters';
import { InvoiceItem } from '../../types/invoice.types';

interface InvoiceItemsTableProps {
  items: InvoiceItem[];
  className?: string;
}

const InvoiceItemsTable: React.FC<InvoiceItemsTableProps> = ({
  items,
  className = '',
}) => {
  const hasProductInfo = items.some(item => item.productId);

  return (
    <div className={`rounded-lg border ${className}`}>
      <Table>
        <TableHeader>
          <TableRow className="bg-muted/50">
            <TableHead className="w-[50px]">S.No</TableHead>
            {hasProductInfo && <TableHead className="w-[100px]">Product</TableHead>}
            <TableHead>Description</TableHead>
            <TableHead className="w-[100px]">HSN/SAC</TableHead>
            <TableHead className="text-right w-[80px]">Qty</TableHead>
            <TableHead className="text-right w-[100px]">Rate</TableHead>
            <TableHead className="text-right w-[120px]">Amount</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody>
          {items.map((item, index) => {
            const hasPriceOverride = item.priceOverride &&
              Math.abs(item.priceOverride.discountPercentage) > 0;
            const hasStockWarning = item.stockWarning;

            return (
              <TableRow key={item.id || index}>
                <TableCell className="font-medium">{index + 1}</TableCell>

                {hasProductInfo && (
                  <TableCell>
                    {item.productSnapshot ? (
                      <TooltipProvider>
                        <Tooltip>
                          <TooltipTrigger>
                            <div className="flex items-center gap-1">
                              <Package className="h-3 w-3 text-muted-foreground" />
                              <Badge variant="outline" className="text-xs">
                                {item.productSnapshot.sku}
                              </Badge>
                            </div>
                          </TooltipTrigger>
                          <TooltipContent>
                            <div className="text-xs space-y-1">
                              <p className="font-semibold">{item.productSnapshot.name}</p>
                              <p>Category: {item.productSnapshot.category}</p>
                              <p>Base Price: {formatCurrency(item.productSnapshot.basePrice)}</p>
                              {hasPriceOverride && (
                                <p className="text-yellow-600">
                                  Price Override: {item.priceOverride?.discountPercentage.toFixed(1)}%
                                </p>
                              )}
                            </div>
                          </TooltipContent>
                        </Tooltip>
                      </TooltipProvider>
                    ) : (
                      <Badge variant="secondary" className="text-xs">Manual</Badge>
                    )}
                  </TableCell>
                )}

                <TableCell>
                  <div className="flex items-center gap-2">
                    {item.description}
                    {hasStockWarning && (
                      <TooltipProvider>
                        <Tooltip>
                          <TooltipTrigger>
                            <AlertTriangle className="h-3 w-3 text-yellow-600" />
                          </TooltipTrigger>
                          <TooltipContent>
                            <p className="text-xs">Low stock warning at time of invoice</p>
                          </TooltipContent>
                        </Tooltip>
                      </TooltipProvider>
                    )}
                  </div>
                  {item.unit && (
                    <span className="text-xs text-muted-foreground ml-1">
                      ({item.unit})
                    </span>
                  )}
                </TableCell>

                <TableCell className="text-muted-foreground">
                  {item.hsnCode || '-'}
                </TableCell>

                <TableCell className="text-right">{item.quantity}</TableCell>

                <TableCell className="text-right">
                  <div>
                    {formatCurrency(item.rate)}
                    {hasPriceOverride && (
                      <div className="text-xs text-muted-foreground line-through">
                        {formatCurrency(item.priceOverride?.originalPrice || 0)}
                      </div>
                    )}
                  </div>
                </TableCell>

                <TableCell className="text-right font-medium">
                  {formatCurrency(item.total)}
                </TableCell>
              </TableRow>
            );
          })}
        </TableBody>
      </Table>
    </div>
  );
};

export default InvoiceItemsTable;
```

### Phase 5: API Service Updates

**File:** `frontend/src/modules/billing/services/invoiceService.ts`

```typescript
// Add product validation to invoice creation
export const invoiceService = {
  // ... existing methods ...

  async createInvoice(invoiceData: InvoiceFormData): Promise<Invoice> {
    try {
      // Validate products if integration is enabled
      if (ENABLE_PRODUCT_INTEGRATION && invoiceData.items.some(item => item.productId)) {
        const productIds = invoiceData.items
          .filter(item => item.productId)
          .map(item => item.productId);

        // Batch validate products
        const validationResponse = await api.post('/products/batch', {
          productIds,
          operation: 'validate'
        });

        if (!validationResponse.data.allValid) {
          throw new Error('Some products are invalid or out of stock');
        }
      }

      // Transform invoice data for API
      const apiData = {
        ...invoiceData,
        items: invoiceData.items.map(item => ({
          ...item,
          // Include product information if available
          productId: item.productId,
          productSnapshot: item.productSnapshot,
          priceOverride: item.priceOverride,
          // Preserve manual entries
          description: item.description,
          hsnCode: item.hsnCode,
          quantity: item.quantity,
          rate: item.rate,
          total: item.total,
          unit: item.unit,
        })),
      };

      const response = await api.post('/invoices', apiData);
      return response.data;
    } catch (error) {
      console.error('Failed to create invoice:', error);
      throw error;
    }
  },

  // Add product search method
  async searchProducts(query: string): Promise<Product[]> {
    try {
      const response = await api.get(`/products/invoice/search`, {
        params: { q: query }
      });
      return response.data;
    } catch (error) {
      console.error('Failed to search products:', error);
      throw error;
    }
  },

  // Add stock validation method
  async validateStock(productId: string, quantity: number): Promise<boolean> {
    try {
      const response = await api.post('/products/validate-stock', {
        productId,
        quantity
      });
      return response.data.available;
    } catch (error) {
      console.error('Failed to validate stock:', error);
      return false;
    }
  },
};
```

## Migration Rollback Strategy

### Environment Variables
```bash
# frontend/.env
# Feature flags for gradual rollout
REACT_APP_PRODUCT_INTEGRATION=true      # Enable product search
REACT_APP_STRICT_MODE=false            # Enforce stock limits
REACT_APP_ALLOW_MANUAL=true            # Allow manual entries
REACT_APP_PRICE_OVERRIDE_THRESHOLD=10  # % threshold for price warnings
```

### Quick Disable
```typescript
// Emergency disable in production
if (window.location.hostname === 'production.domain.com') {
  window.DISABLE_PRODUCT_INTEGRATION = true;
}
```

## Testing Checklist

### Creation Flow Tests
- [ ] Product search works with 2+ characters
- [ ] Debouncing prevents excessive API calls
- [ ] Products auto-populate all fields correctly
- [ ] Manual entry still works when needed
- [ ] Quantity validation shows warnings
- [ ] Price override tracking works
- [ ] Tax calculations remain accurate
- [ ] Stock warnings appear correctly
- [ ] Form validation handles mixed items (product + manual)
- [ ] Invoice saves with product data

### View Flow Tests
- [ ] Product badges display correctly
- [ ] SKU information shows in tooltips
- [ ] Price overrides are highlighted
- [ ] Stock warnings are visible
- [ ] Legacy invoices display without errors
- [ ] Mixed invoices (product + manual) render properly
- [ ] Print view includes product information
- [ ] PDF generation works with new fields

### API Integration Tests
- [ ] Product search API returns results
- [ ] Stock validation prevents over-ordering
- [ ] Batch product validation works
- [ ] Invoice creation includes product data
- [ ] Product snapshots are preserved
- [ ] Price override data is stored
- [ ] Backend accepts new invoice format
- [ ] Legacy invoice format still works

## Migration Timeline

### Week 1-2: Development
- Implement type updates
- Create ProductSearchField component
- Update LineItemsSection
- Modify invoice view components
- Test in development environment

### Week 3: Staging
- Deploy to staging with feature flags OFF
- Enable for select test users
- Monitor performance and errors
- Gather feedback

### Week 4: Soft Launch
- Enable product search for all staging users
- Keep strict mode OFF
- Allow manual entries
- Monitor adoption rate

### Week 5-6: Production Rollout
- Deploy to production with feature flags OFF
- Gradually enable by percentage
- Monitor error rates
- Provide user training

### Week 7-8: Full Migration
- Enable for all users
- Consider enabling strict mode
- Phase out manual-only entries
- Complete documentation

## Key Benefits

1. **Data Integrity**: Product information is consistent across invoices
2. **Efficiency**: Faster invoice creation with auto-fill
3. **Inventory Control**: Real-time stock validation
4. **Price Management**: Automatic tracking of discounts
5. **Audit Trail**: Complete history of product usage
6. **Error Reduction**: Less manual data entry errors
7. **Analytics**: Better insights into product performance

## Support and Troubleshooting

### Common Issues
1. **Product not found**: Check product catalog, ensure active status
2. **Stock warnings**: Verify actual stock levels in inventory
3. **Price mismatches**: Review price override settings
4. **Search not working**: Check network, API availability
5. **Legacy invoices**: Ensure backward compatibility code is active

### Debug Mode
```typescript
// Enable debug logging
localStorage.setItem('DEBUG_INVOICE_PRODUCTS', 'true');
```

### Support Contacts
- Technical Issues: tech-support@company.com
- Training: training@company.com
- Feedback: product-feedback@company.com